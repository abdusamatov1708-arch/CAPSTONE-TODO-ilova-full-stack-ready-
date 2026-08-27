# CAPSTONE-TODO-ilova-full-stack-ready-
Modul Kodlari
model.js (Domain Model)
JavaScript
export class Todo {
    #id;
    #text;
    #completed;
    #important;
    #createdAt;

    constructor({ id, text, completed = false, important = false, createdAt }) {
        this.#id = id || crypto.randomUUID();
        this.#text = text;
        this.#completed = completed;
        this.#important = important;
        this.#createdAt = createdAt || new Date().toISOString();
    }

    get id() { return this.#id; }
    get text() { return this.#text; }
    get completed() { return this.#completed; }
    get important() { return this.#important; }
    get createdAt() { return this.#createdAt; }

    toggleComplete() {
        this.#completed = !this.#completed;
    }

    toggleImportant() {
        this.#important = !this.#important;
    }

    updateText(newText) {
        if (!newText.trim()) throw new Error("Matn bo'sh bo'lishi mumkin emas!");
        this.#text = newText.trim();
    }

    // JSON ga o'tkazish uchun yordamchi
    toJSON() {
        return {
            id: this.#id,
            text: this.#text,
            completed: this.#completed,
            important: this.#important,
            createdAt: this.#createdAt
        };
    }
}
store.js (State Management, Observer & localStorage)
JavaScript
import { Todo } from './model.js';

const STORAGE_KEY = 'todo_app_v1';

export class TodoStore {
    #todos;
    #listeners;

    constructor() {
        this.#todos = [];
        this.#listeners = new Set();
        this.#load();
    }

    #load() {
        try {
            const data = localStorage.getItem(STORAGE_KEY);
            if (data) {
                const parsed = JSON.parse(data);
                this.#todos = parsed.map(item => new Todo(item));
            }
        } catch (error) {
            console.error("Keshdan o'qishda xatolik:", error);
            this.#todos = [];
        }
    }

    #save() {
        try {
            const data = this.#todos.map(t => t.toJSON());
            localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
            this.#notify();
        } catch (error) {
            console.error("Keshga yozishda xatolik:", error);
        }
    }

    #notify() {
        this.#listeners.forEach(listener => listener(this));
    }

    // Observer: Subscribe / Unsubscribe
    subscribe(fn) {
        this.#listeners.add(fn);
        return () => this.#listeners.delete(fn);
    }

    // --- Computed Getters ---
    get all() {
        return [...this.#todos];
    }

    get active() {
        return this.#todos.filter(t => !t.completed);
    }

    get completed() {
        return this.#todos.filter(t => t.completed);
    }

    get important() {
        return this.#todos.filter(t => t.important);
    }

    get stats() {
        const total = this.#todos.length;
        const done = this.#todos.filter(t => t.completed).length;
        return {
            total,
            done,
            active: total - done,
            percent: total === 0 ? 0 : Math.round((done / total) * 100)
        };
    }

    // --- Actions ---
    add(text) {
        const newTodo = new Todo({ text });
        this.#todos.unshift(newTodo);
        this.#save();
        return newTodo;
    }

    remove(id) {
        this.#todos = this.#todos.filter(t => t.id !== id);
        this.#save();
    }

    toggleComplete(id) {
        const todo = this.#todos.find(t => t.id === id);
        if (todo) {
            todo.toggleComplete();
            this.#save();
        }
    }

    toggleImportant(id) {
        const todo = this.#todos.find(t => t.id === id);
        if (todo) {
            todo.toggleImportant();
            this.#save();
        }
    }

    importJSON(jsonString) {
        try {
            const parsed = JSON.parse(jsonString);
            if (!Array.isArray(parsed)) throw new Error("Format noto'g'ri");
            this.#todos = parsed.map(item => new Todo(item));
            this.#save();
            return true;
        } catch (e) {
            console.error("Import xatosi:", e);
            return false;
        }
    }
}
api.js (Server Sync Mock)
JavaScript
export const TodoApi = {
    async syncWithServer(todos) {
        // Real loyihada fetch('/api/todos', { method: 'POST', body: JSON.stringify(todos) }) bo'ladi
        return new Promise((resolve, reject) => {
            setTimeout(() => {
                if (Math.random() < 0.05) {
                    reject(new Error("Server bilan sinxronizatsiyada xatolik!"));
                } else {
                    resolve({ success: true, syncedAt: new Date().toISOString() });
                }
            }, 800);
        });
    }
};
app.js (DOM Render, Event Delegation & Debounce)
JavaScript
import { TodoStore } from './store.js';
import { TodoApi } from './api.js';

const store = new TodoStore();

// UI Elementlari
const formEl = document.getElementById('todo-form');
const inputEl = document.getElementById('todo-input');
const listEl = document.getElementById('todo-list');
const searchEl = document.getElementById('search-input');
const filterBtns = document.querySelectorAll('.filter-btn');
const statsEl = document.getElementById('stats-text');
const syncBtn = document.getElementById('sync-btn');
const exportBtn = document.getElementById('export-btn');
const themeBtn = document.getElementById('theme-btn');

let currentFilter = 'all'; // 'all' | 'active' | 'important' | 'completed'
let searchQuery = '';

// --- Debounce funksiyasi (Qidirish uchun) ---
function debounce(fn, delay) {
    let timeoutId;
    return (...args) => {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => fn(...args), delay);
    };
}

// --- Render funksiyasi ---
function render() {
    let items = [];
    switch (currentFilter) {
        case 'active': items = store.active; break;
        case 'important': items = store.important; break;
        case 'completed': items = store.completed; break;
        default: items = store.all;
    }

    // Qidiruv filteri
    if (searchQuery.trim()) {
        const q = searchQuery.toLowerCase();
        items = items.filter(t => t.text.toLowerCase().includes(q));
    }

    // DOM ga chiqarish (Template literallar bilan)
    listEl.innerHTML = items.length === 0 
        ? `<li class="empty">Topshiriqlar topilmadi</li>`
        : items.map(todo => `
            <li data-id="${todo.id}" class="${todo.completed ? 'completed' : ''} ${todo.important ? 'important' : ''}">
                <input type="checkbox" class="toggle" ${todo.completed ? 'checked' : ''}>
                <span class="text">${escapeHtml(todo.text)}</span>
                <div class="actions">
                    <button class="btn-star" title="Muhim">${todo.important ? '⭐' : '☆'}</button>
                    <button class="btn-delete" title="O'chirish">🗑️</button>
                </div>
            </li>
        `).join('');

    // Statistikani yangilash
    const stats = store.stats;
    statsEl.textContent = `Jami: ${stats.total} | Bajarilgan: ${stats.done} (${stats.percent}%)`;
}

// X XSS himoyasi uchun yordamchi
function escapeHtml(str) {
    return str.replace(/[&<>'"]/g, 
        tag => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', "'": '&#39;', '"': '&quot;' }[tag] || tag)
    );
}

// --- Event Listeners ---

// 1. Yangi todo qo'shish (Enter yoki Submit)
formEl.addEventListener('submit', (e) => {
    e.preventDefault();
    try {
        store.add(inputEl.value);
        inputEl.value = '';
    } catch (err) {
        alert(err.message);
    }
});

// 2. Event Delegation (Barcha LI hodisalarini bitta joyda boshqarish)
listEl.addEventListener('click', (e) => {
    const li = e.target.closest('li');
    if (!li) return;
    const id = li.dataset.id;

    if (e.target.classList.contains('toggle')) {
        store.toggleComplete(id);
    } else if (e.target.classList.contains('btn-star')) {
        store.toggleImportant(id);
    } else if (e.target.classList.contains('btn-delete')) {
        store.remove(id);
    }
});

// 3. Debounce qidirish
const handleSearch = debounce((e) => {
    searchQuery = e.target.value;
    render();
}, 300);

searchEl.addEventListener('input', handleSearch);

// 4. Filter tugmalari
filterBtns.forEach(btn => {
    btn.addEventListener('click', () => {
        filterBtns.forEach(b => b.classList.remove('active'));
        btn.classList.add('active');
        currentFilter = btn.dataset.filter;
        render();
    });
});

// 5. Server Sync (Async/Await)
syncBtn.addEventListener('click', async () => {
    syncBtn.textContent = "Sinxronlanmoqda...";
    syncBtn.disabled = true;
    try {
        await TodoApi.syncWithServer(store.all);
        alert("Server bilan muvaffaqiyatli sinxronlandi! ✅");
    } catch (err) {
        alert(err.message + " ❌");
    } finally {
        syncBtn.textContent = "Serverga Sinxronlash";
        syncBtn.disabled = false;
    }
});

// 6. Bonus: JSON eksport
exportBtn.addEventListener('click', () => {
    const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(store.all, null, 2));
    const dlAnchor = document.createElement('a');
    dlAnchor.setAttribute("href", dataStr);
    dlAnchor.setAttribute("download", `todos_${new Date().toISOString().split('T')[0]}.json`);
    document.body.appendChild(dlAnchor);
    dlAnchor.click();
    dlAnchor.remove();
});

// 7. Bonus: Dark Mode
themeBtn.addEventListener('click', () => {
    document.body.classList.toggle('dark-theme');
    themeBtn.textContent = document.body.classList.contains('dark-theme') ? '☀️ Light' : '🌙 Dark';
});

// Observer orqali state o'zgarganda avtomatik render qilish
store.subscribe(() => {
    render();
});

// Boshlang'ich render
render();
3. HTML Sahifa (index.html)
HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Advanced Capstone TODO App</title>
    <style>
        body { font-family: sans-serif; max-width: 600px; margin: 30px auto; padding: 0 15px; transition: background 0.3s, color 0.3s; }
        body.dark-theme { background: #181818; color: #f0f0f0; }
        .controls, .filters { display: flex; gap: 10px; margin-bottom: 15px; flex-wrap: wrap; }
        input[type="text"] { flex: 1; padding: 8px; }
        button { padding: 8px 12px; cursor: pointer; }
        ul { list-style: none; padding: 0; }
        li { display: flex; align-items: center; justify-content: space-between; padding: 10px; border: 1px solid #ddd; margin-bottom: 6px; border-radius: 4px; }
        li.completed span.text { text-decoration: line-through; color: gray; }
        li.important { border-color: orange; background: rgba(255, 165, 0, 0.05); }
        .actions { display: flex; gap: 5px; }
        .filter-btn.active { background: #007bff; color: white; border-color: #007bff; }
    </style>
</head>
<body>

    <div style="display: flex; justify-content: space-between; align-items: center;">
        <h2>TODO Kapstoun</h2>
        <div>
            <button id="theme-btn">🌙 Dark</button>
            <button id="sync-btn">Serverga Sinxronlash</button>
            <button id="export-btn">JSON Eksport</button>
        </div>
    </div>

    <!-- Yangi todo qo'shish formasi -->
    <form id="todo-form" class="controls">
        <input type="text" id="todo-input" placeholder="Yangi vazifa kiriting va Enter bosing..." required>
        <button type="submit">Qo'shish</button>
    </form>

    <!-- Qidirish va Filtrlar -->
    <div class="controls">
        <input type="text" id="search-input" placeholder="🔍 Qidirish (debounce bilan)...">
    </div>

    <div class="filters">
        <button class="filter-btn active" data-filter="all">Barchasi</button>
        <button class="filter-btn" data-filter="active">Bajarilmagan</button>
        <button class="filter-btn" data-filter="important">Muhim ⭐</button>
        <button class="filter-btn" data-filter="completed">Bajarilgan</button>
    </div>

    <p id="stats-text" style="font-weight: bold; font-size: 14px;"></p>
    <hr>

    <!-- Vazifalar ro'yxati -->
    <ul id="todo-list"></ul>

    <!-- ES6 Modul sifatida ulash -->
    <script type="module" src="./js/app.js"></script>
</body>
</html>
