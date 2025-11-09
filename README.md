<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Matriz de Eisenhower Pro</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
    <style>
        @keyframes spin { to { transform: rotate(360deg); } }
        .animate-spin { animation: spin 1s linear infinite; }
    </style>
</head>
<body class="bg-gradient-to-br from-slate-50 to-blue-50 min-h-screen">
    <div id="app"></div>
    <script>
        const STORAGE_KEY = 'eisenhower-tasks-v2';
        const app = {
            tasks: { urgent_important: [], not_urgent_important: [], urgent_not_important: [], not_urgent_not_important: [] },
            view: 'matrix', activeTimer: null, timerSeconds: 0, isRunning: false,
            showAddModal: false, showAIModal: false, selectedQuadrant: null,
            newTaskText: '', newTaskTime: 30, userInput: '', uploadedFile: null, isClassifying: false,
            quadrants: {
                urgent_important: { title: 'HACER AHORA', subtitle: 'Urgente e Importante', color: 'from-red-500 to-orange-500', bgColor: 'bg-red-50', borderColor: 'border-red-200', advice: 'Crisis, deadlines, emergencias', priority: 1 },
                not_urgent_important: { title: 'PLANIFICAR', subtitle: 'No Urgente pero Importante', color: 'from-blue-500 to-indigo-500', bgColor: 'bg-blue-50', borderColor: 'border-blue-200', advice: 'Planificación, desarrollo personal', priority: 2 },
                urgent_not_important: { title: 'DELEGAR', subtitle: 'Urgente pero No Importante', color: 'from-yellow-500 to-amber-500', bgColor: 'bg-yellow-50', borderColor: 'border-yellow-200', advice: 'Interrupciones, algunas llamadas', priority: 3 },
                not_urgent_not_important: { title: 'ELIMINAR', subtitle: 'Ni Urgente ni Importante', color: 'from-gray-500 to-slate-500', bgColor: 'bg-gray-50', borderColor: 'border-gray-200', advice: 'Distracciones, redes sociales', priority: 4 }
            },
            init() { this.loadData(); this.render(); this.startTimerInterval(); },
            loadData() { const saved = localStorage.getItem(STORAGE_KEY); if (saved) { const data = JSON.parse(saved); this.tasks = data.tasks || this.tasks; this.activeTimer = data.activeTimer; } },
            saveData() { localStorage.setItem(STORAGE_KEY, JSON.stringify({ tasks: this.tasks, activeTimer: this.activeTimer })); },
            startTimerInterval() { setInterval(() => { if (this.isRunning) { this.timerSeconds++; this.render(); } }, 1000); },
            addTask(quadrant, text, time = 0) { this.tasks[quadrant].push({ id: Date.now() + Math.random(), text, estimatedTime: time, timeSpent: 0, completed: false }); this.saveData(); this.render(); },
            deleteTask(quadrant, taskId) { this.tasks[quadrant] = this.tasks[quadrant].filter(t => t.id !== taskId); if (this.activeTimer?.quadrant === quadrant && this.activeTimer?.taskId === taskId) { this.activeTimer = null; this.isRunning = false; this.timerSeconds = 0; } this.saveData(); this.render(); },
            toggleComplete(quadrant, taskId) { const task = this.tasks[quadrant].find(t => t.id === taskId); if (task) { task.completed = !task.completed; this.saveData(); this.render(); } },
            startTimer(quadrant, taskId) { if (this.activeTimer?.quadrant !== quadrant || this.activeTimer?.taskId !== taskId) { if (this.activeTimer && this.isRunning) this.pauseTimer(); this.activeTimer = { quadrant, taskId }; this.timerSeconds = 0; } this.isRunning = true; this.render(); },
            pauseTimer() { this.isRunning = false; if (this.activeTimer) { const task = this.tasks[this.activeTimer.quadrant].find(t => t.id === this.activeTimer.taskId); if (task) { task.timeSpent += this.timerSeconds; this.saveData(); } } this.render(); },
            resetTimer() { this.isRunning = false; this.timerSeconds = 0; this.render(); },
            formatTime(seconds) { const hrs = Math.floor(seconds / 3600); const mins = Math.floor((seconds % 3600) / 60); const secs = seconds % 60; return `${String(hrs).padStart(2, '0')}:${String(mins).padStart(2, '0')}:${String(secs).padStart(2, '0')}`; },
            getStats() { let total = 0, completed = 0, totalTime = 0; Object.values(this.tasks).forEach(arr => { total += arr.length; completed += arr.filter(t => t.completed).length; totalTime += arr.reduce((acc, t) => acc + t.timeSpent, 0); }); totalTime += (this.activeTimer ? this.timerSeconds : 0); return { total, completed, pending: total - completed, totalTime }; },
            async classifyWithAI() {
                if (!this.userInput.trim() && !this.uploadedFile) return;
                this.isClassifying = true; this.render();
                try {
                    let textToClassify = this.userInput;
                    if (this.uploadedFile) {
                        if (this.uploadedFile.type === 'text/plain') { textToClassify += '\n\n' + await this.uploadedFile.text(); }
                        else { const arrayBuffer = await this.uploadedFile.arrayBuffer(); const workbook = XLSX.read(arrayBuffer, { type: 'array' }); const firstSheet = workbook.Sheets[workbook.SheetNames[0]]; const data = XLSX.utils.sheet_to_json(firstSheet, { header: 1 }); const excelText = data.filter(row => row && row.length > 0).map(row => row.join(' - ')).join('\n'); textToClassify += '\n\n' + excelText; }
                    }
                    const response = await fetch("https://api.anthropic.com/v1/messages", {
                        method: "POST", headers: { "Content-Type": "application/json" },
                        body: JSON.stringify({ model: "claude-sonnet-4-20250514", max_tokens: 1000, messages: [{ role: "user", content: `Clasifica estas tareas en la Matriz de Eisenhower: 1. urgent_important (Urgente e Importante) 2. not_urgent_important (No Urgente pero Importante) 3. urgent_not_important (Urgente pero No Importante) 4. not_urgent_not_important (Ni Urgente ni Importante). Usuario: "${textToClassify}". Responde SOLO con JSON válido sin markdown: {"tasks": [{"text": "descripción", "quadrant": "urgent_important"}]}` }] })
                    });
                    const data = await response.json(); const content = data.content[0].text; const result = JSON.parse(content.replace(/```json|```/g, '').trim());
                    if (result.tasks) { result.tasks.forEach(task => { if (task.quadrant && this.tasks[task.quadrant]) this.addTask(task.quadrant, task.text, 0); }); }
                    this.showAIModal = false; this.userInput = ''; this.uploadedFile = null;
                } catch (error) { alert('Error al clasificar. Verifica el archivo o intenta de nuevo.'); }
                this.isClassifying = false; this.render();
            },
            render() {
                const stats = this.getStats(); const activeTask = this.activeTimer ? this.tasks[this.activeTimer.quadrant]?.find(t => t.id === this.activeTimer.taskId) : null;
                document.getElementById('app').innerHTML = `<div class="max-w-7xl mx-auto p-4 md:p-6">
                    <div class="bg-white rounded-2xl shadow-lg p-6 mb-6">
                        <div class="flex flex-col md:flex-row items-start md:items-center justify-between gap-4 mb-4">
                            <div><h1 class="text-3xl font-bold text-gray-800 mb-1">Matriz de Eisenhower Pro <span class="text-sm font-normal text-green-600 ml-2">💾 Guardado auto</span></h1><p class="text-gray-600 text-sm">Organiza con IA y timer</p></div>
                            <div class="flex flex-wrap gap-2">
                                <button onclick="app.showAIModal = true; app.render();" class="px-4 py-2 rounded-lg font-medium bg-gradient-to-r from-purple-600 to-pink-600 text-white hover:from-purple-700 hover:to-pink-700">✨ Clasificar con IA</button>
                                <button onclick="app.view = 'matrix'; app.render();" class="px-4 py-2 rounded-lg font-medium ${app.view === 'matrix' ? 'bg-indigo-600 text-white' : 'bg-gray-100 text-gray-700'}">Matriz</button>
                                <button onclick="app.view = 'list'; app.render();" class="px-4 py-2 rounded-lg font-medium ${app.view === 'list' ? 'bg-indigo-600 text-white' : 'bg-gray-100 text-gray-700'}">Lista</button>
                            </div>
                        </div>
                        <div class="grid grid-cols-2 md:grid-cols-4 gap-3">
                            <div class="bg-indigo-50 rounded-lg p-3 text-center"><div class="text-2xl font-bold text-indigo-600">${stats.total}</div><div class="text-xs text-gray-600">Total</div></div>
                            <div class="bg-green-50 rounded-lg p-3 text-center"><div class="text-2xl font-bold text-green-600">${stats.completed}</div><div class="text-xs text-gray-600">Completadas</div></div>
                            <div class="bg-yellow-50 rounded-lg p-3 text-center"><div class="text-2xl font-bold text-yellow-600">${stats.pending}</div><div class="text-xs text-gray-600">Pendientes</div></div>
                            <div class="bg-purple-50 rounded-lg p-3 text-center"><div class="text-2xl font-bold text-purple-600">${this.formatTime(stats.totalTime)}</div><div class="text-xs text-gray-600">Tiempo Total</div></div>
                        </div>
                    </div>
                    ${activeTask ? this.renderActiveTimer(activeTask) : ''}
                    ${this.view === 'matrix' ? this.renderMatrix() : this.renderList()}
                    ${this.showAddModal ? this.renderAddModal() : ''}
                    ${this.showAIModal ? this.renderAIModal() : ''}
                </div>`;
            },
            renderActiveTimer(task) {
                const quad = this.quadrants[this.activeTimer.quadrant];
                return `<div class="bg-gradient-to-r ${quad.color} rounded-2xl p-6 mb-6 text-white shadow-lg"><div class="text-center">
                    <div class="text-sm opacity-90 mb-2">Trabajando en: ${quad.title}</div><div class="text-lg font-semibold mb-4">${task.text}</div>
                    <div class="text-5xl font-bold mb-4">${this.formatTime(this.timerSeconds)}</div>
                    <div class="flex justify-center gap-3">
                        ${!this.isRunning ? `<button onclick="app.isRunning = true; app.render();" class="px-6 py-3 bg-white/20 rounded-lg hover:bg-white/30">▶️ Iniciar</button>` : `<button onclick="app.pauseTimer();" class="px-6 py-3 bg-white/20 rounded-lg hover:bg-white/30">⏸️ Pausar</button>`}
                        <button onclick="app.resetTimer();" class="px-6 py-3 bg-white/10 rounded-lg hover:bg-white/20">🔄 Reiniciar</button>
                    </div></div></div>`;
            },
            renderMatrix() {
                return `<div class="grid grid-cols-1 md:grid-cols-2 gap-4">${Object.entries(this.quadrants).map(([key, quad]) => `
                    <div class="${quad.bgColor} border-2 ${quad.borderColor} rounded-2xl shadow-lg">
                        <div class="bg-gradient-to-r ${quad.color} text-white p-4"><h3 class="text-xl font-bold">${quad.title}</h3><p class="text-xs opacity-90">${quad.subtitle}</p></div>
                        <div class="p-4">
                            <div class="space-y-2 mb-4 max-h-80 overflow-y-auto">
                                ${this.tasks[key].length === 0 ? '<div class="text-center py-8 text-gray-400 text-sm">Sin tareas</div>' : this.tasks[key].map(task => `
                                    <div class="bg-white rounded-lg p-3 shadow-sm border ${task.completed ? 'opacity-60' : ''}">
                                        <div class="flex items-start gap-2 mb-2">
                                            <button onclick="app.toggleComplete('${key}', ${task.id})">${task.completed ? '<span class="text-green-500">✓</span>' : '<span class="text-gray-400">○</span>'}</button>
                                            <p class="text-sm flex-1 ${task.completed ? 'line-through text-gray-500' : ''}">${task.text}</p>
                                        </div>
                                        <div class="flex items-center justify-between ml-7">
                                            <span class="text-xs text-gray-500">${task.estimatedTime > 0 ? `⏱️ ${task.estimatedTime}min` : '⏱️ Sin tiempo'}</span>
                                            <div class="flex gap-1">
                                                ${!task.completed ? `<button onclick="app.startTimer('${key}', ${task.id})" class="p-1 text-indigo-600 hover:bg-indigo-50 rounded text-sm">▶️</button>` : ''}
                                                <button onclick="app.deleteTask('${key}', ${task.id})" class="p-1 text-red-500 hover:bg-red-50 rounded text-sm">🗑️</button>
                                            </div>
                                        </div>
                                    </div>
                                `).join('')}
                            </div>
                            <button onclick="app.selectedQuadrant = '${key}'; app.showAddModal = true; app.render();" class="w-full py-2 bg-white border-2 border-dashed border-gray-300 rounded-lg text-gray-600 hover:border-gray-400 text-sm">➕ Agregar</button>
                        </div>
                    </div>
                `).join('')}</div>`;
            },
            renderList() {
                const sorted = []; Object.entries(this.quadrants).forEach(([key, quad]) => { this.tasks[key].forEach(task => { sorted.push({ ...task, quadrant: key, quad }); }); });
                sorted.sort((a, b) => { if (a.completed !== b.completed) return a.completed ? 1 : -1; return a.quad.priority - b.quad.priority; });
                return `<div class="bg-white rounded-2xl shadow-lg p-6"><h2 class="text-2xl font-bold mb-4">📋 Por Prioridad</h2>
                    ${sorted.length === 0 ? '<div class="text-center py-12 text-gray-400">Sin tareas</div>' : sorted.map((task, i) => `
                        <div class="flex items-center gap-3 p-3 bg-gray-50 rounded-lg mb-2">
                            <div class="w-8 h-8 rounded-full bg-gradient-to-r ${task.quad.color} text-white flex items-center justify-center font-bold text-sm">${i + 1}</div>
                            <div class="flex-1"><p class="font-medium">${task.text}</p><p class="text-xs text-gray-500">${task.quad.title}</p></div>
                            <button onclick="app.deleteTask('${task.quadrant}', ${task.id})" class="p-2 text-red-500 hover:bg-red-50 rounded">🗑️</button>
                        </div>
                    `).join('')}</div>`;
            },
            renderAddModal() {
                return `<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
                    <div class="bg-white rounded-2xl shadow-2xl max-w-md w-full p-6">
                        <h3 class="text-xl font-bold mb-4">${this.quadrants[this.selectedQuadrant].title}</h3>
                        <input id="taskInput" type="text" placeholder="Describe la tarea..." class="w-full px-4 py-3 border-2 rounded-lg mb-4">
                        <div class="mb-4"><label class="block text-sm font-medium mb-2">Tiempo estimado (min) - Opcional</label>
                        <input id="timeInput" type="number" value="30" class="w-full px-4 py-2 border-2 rounded-lg"></div>
                        <div class="flex gap-3">
                            <button onclick="const text = document.getElementById('taskInput').value; const time = parseInt(document.getElementById('timeInput').value) || 0; if (text) { app.addTask(app.selectedQuadrant, text, time); app.showAddModal = false; }" class="flex-1 bg-indigo-600 text-white py-3 rounded-lg font-semibold">Agregar</button>
                            <button onclick="app.showAddModal = false; app.render();" class="flex-1 bg-gray-200 py-3 rounded-lg font-semibold">Cancelar</button>
                        </div>
                    </div>
                </div>`;
            },
            renderAIModal() {
                return `<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
                    <div class="bg-white rounded-2xl shadow-2xl max-w-2xl w-full p-6">
                        <h3 class="text-2xl font-bold mb-2">✨ Clasificación Inteligente</h3><p class="text-sm text-gray-600 mb-4">Cuéntame tus tareas o sube un archivo</p>
                        <div class="bg-purple-50 rounded-lg p-3 mb-4"><p class="text-xs text-gray-700"><strong>Ejemplo:</strong> "Tengo que entregar un proyecto mañana, quiero hacer ejercicio, me interrumpen con llamadas"</p></div>
                        <textarea id="aiInput" placeholder="Escribe todas tus tareas aquí..." class="w-full px-4 py-3 border-2 rounded-lg mb-4 h-32" ${this.isClassifying ? 'disabled' : ''}>${this.userInput}</textarea>
                        <div class="mb-4"><label class="block text-sm font-medium mb-2">📎 O sube un archivo (.txt, .xlsx, .xls)</label>
                        <input id="fileInput" type="file" accept=".txt,.xlsx,.xls" ${this.isClassifying ? 'disabled' : ''} onchange="app.uploadedFile = this.files[0]; app.render();" class="w-full px-4 py-2 border-2 border-dashed border-gray-300 rounded-lg text-sm">
                        ${this.uploadedFile ? `<p class="text-sm text-green-600 mt-2">✓ ${this.uploadedFile.name}</p>` : ''}</div>
                        <div class="bg-blue-50 rounded-lg p-3 mb-4"><p class="text-xs text-gray-700"><strong>💡:</strong> Las tareas "DELEGAR" se agregan automáticamente</p></div>
                        <div class="flex gap-3">
                            <button onclick="app.userInput = document.getElementById('aiInput').value; app.classifyWithAI();" ${this.isClassifying ? 'disabled' : ''} class="flex-1 bg-gradient-to-r from-purple-600 to-pink-600 text-white py-3 rounded-lg font-semibold disabled:opacity-50">${this.isClassifying ? '⏳ Clasificando...' : '✨ Clasificar'}</button>
                            <button onclick="app.showAIModal = false; app.userInput = ''; app.uploadedFile = null; app.render();" ${this.isClassifying ? 'disabled' : ''} class="flex-1 bg-gray-200 py-3 rounded-lg font-semibold disabled:opacity-50">Cancelar</button>
                        </div>
                    </div>
                </div>`;
            }
        };
        app.init();
    </script>
</body>
</html>
