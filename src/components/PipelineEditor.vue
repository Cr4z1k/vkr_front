<template>
  <div class="pipeline-wrapper">
    <div class="toolbar">
      <button @click="addNode">Добавить обработчик</button>
      <button @click="exportPipeline" :disabled="isCyclic || hasInvalidYaml" :class="{ disabled: isCyclic || hasInvalidYaml }">Экспортировать и вывести JSON</button>
      <button @click="sendToServer" :disabled="isCyclic || hasInvalidYaml" :class="{ disabled: isCyclic || hasInvalidYaml }">Отправить на сервер</button>
    </div>
    <div v-if="isCyclic" class="cyclic-warning">
      В графе обнаружен цикл! Экспорт невозможен.
    </div>
    <div v-if="hasInvalidYaml" class="cyclic-warning invalid-yaml-warning">
      В одном или нескольких обработчиках некорректный YAML! Экспорт невозможен.
    </div>
    <VueFlow
      v-model:nodes="nodes"
      v-model:edges="edges"
      :fit-view="true"
      :min-zoom="0.2"
      :max-zoom="2"
      class="pipeline-flow"
      :connection-mode="'loose'"
      :connectable="true"
      :elements-connectable="true"
      :nodes-connectable="true"
      :edges-updatable="true"
      :edges-deletable="true"
      @connect="onConnect"
      @edge-update="onEdgeUpdate"
      @edge-delete="onEdgeDelete"
    >
      <template #node-processor="{ id, data }">
        <div class="processor-node" @dblclick="openEditor(id)" :class="{ 'invalid-yaml': invalidYamlNodes.includes(id), 'server-error': serverErrorNodes.includes(id) }">
          <Handle type="target" :position="Position.Left" :id="'in'" />
          <div class="node-header">{{ id }}</div>
          <div class="node-config-preview">
            <span class="config-empty">Двойной клик для редактирования</span>
          </div>
          <Handle type="source" :position="Position.Right" :id="'out'" />
        </div>
      </template>
      <Controls />
      <MiniMap />
    </VueFlow>

    <!-- Модальное окно -->
    <template v-if="editorNodeId">
      <div class="modal-backdrop" @click.self="closeEditor">
        <div class="modal-editor">
          <h3>Редактирование {{ editorNodeId }}</h3>
          <label>Config (Redpanda Connect):</label>
          <textarea v-model="editorNode.data.config" placeholder="Config (Redpanda Connect)"></textarea>
          <label v-if="isInputNode(editorNodeId)">Input config:</label>
          <textarea v-if="isInputNode(editorNodeId)" v-model="editorNode.data.input" placeholder="Input config"></textarea>
          <label v-if="isOutputNode(editorNodeId)">Output config:</label>
          <textarea v-if="isOutputNode(editorNodeId)" v-model="editorNode.data.output" placeholder="Output config"></textarea>
          <div class="modal-actions">
            <button @click="closeEditor">Сохранить</button>
          </div>
        </div>
      </div>
    </template>

    <!-- Экспортированный JSON -->
    <div v-if="exportedJson" class="exported-json-view">
      <div class="exported-json-header">
        <h4>Экспортированный JSON:</h4>
        <button class="close-json" @click="exportedJson = ''">×</button>
      </div>
      <pre>{{ exportedJson }}</pre>
    </div>

    <!-- Статус отправки -->
    <div v-if="sendStatus" :class="['send-status', sendStatus === 'success' ? 'success' : 'error']">
      {{ sendMessage }}
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';
import { VueFlow, Handle, Position } from '@vue-flow/core';
import { MiniMap } from '@vue-flow/minimap';
import { Controls } from '@vue-flow/controls';
import axios from 'axios';
import yaml from 'js-yaml';

const nodes = ref([
  {
    id: 'step1',
    type: 'processor',
    position: { x: 100, y: 100 },
    data: { config: '', input: '', output: '' },
  },
]);
const edges = ref([]);
let nodeId = 2;
const editorNodeId = ref(null);
const editorNode = ref(null);
const exportedJson = ref('');
const sendStatus = ref(''); // '', 'success', 'error'
const sendMessage = ref('');
const serverErrorNodes = ref([]); // id нод с ошибкой сервера

function addNode() {
  nodes.value.push({
    id: `step${nodeId++}`,
    type: 'processor',
    position: { x: 100 + 80 * nodeId, y: 100 + 40 * nodeId },
    data: { config: '', input: '', output: '' },
  });
}

function isInputNode(id) {
  // Нет входящих связей
  return !edges.value.some(e => e.target === id);
}
function isOutputNode(id) {
  // Нет исходящих связей
  return !edges.value.some(e => e.source === id);
}

function cleanInputOutput(str, prefix) {
  if (!str) return '';
  const trimmed = str.trim();
  if (trimmed.startsWith(prefix)) {
    return trimmed.slice(prefix.length).trimStart();
  }
  return trimmed;
}
function cleanConfig(str) {
  if (!str) return '';
  const trimmed = str.trim();
  if (trimmed.startsWith('pipeline:')) {
    return trimmed.slice('pipeline:'.length).trimStart();
  }
  return trimmed;
}

function exportPipeline() {
  // 1. Строим неориентированный граф
  const nodeMap = {};
  nodes.value.forEach(n => { nodeMap[n.id] = n; });
  const adj = {};
  nodes.value.forEach(n => { adj[n.id] = []; });
  edges.value.forEach(e => {
    if (adj[e.source] && !adj[e.source].includes(e.target)) adj[e.source].push(e.target);
    if (adj[e.target] && !adj[e.target].includes(e.source)) adj[e.target].push(e.source);
  });

  // 2. BFS/DFS для поиска компонент связности
  const visited = new Set();
  const components = [];
  for (const n of nodes.value) {
    if (!visited.has(n.id)) {
      const queue = [n.id];
      const compNodes = [];
      while (queue.length) {
        const curr = queue.pop();
        if (visited.has(curr)) continue;
        visited.add(curr);
        compNodes.push(curr);
        for (const nb of adj[curr]) {
          if (!visited.has(nb)) queue.push(nb);
        }
      }
      components.push(compNodes);
    }
  }

  // 3. Для каждой компоненты собираем nodes и edges
  const pipelines = components.map((comp, idx) => {
    // Собираем только связи внутри компоненты
    const compEdges = edges.value.filter(e => comp.includes(e.source) && comp.includes(e.target));
    // Собираем ноды
    const compNodes = comp.map(id => {
      const n = nodeMap[id];
      const isSink = !compEdges.some(e => e.source === id);
      const node = {
        id: n.id,
        type: isSink ? 'sink' : 'processor',
        config: cleanConfig(n.data.config),
      };
      if (!compEdges.some(e => e.target === n.id) && n.data.input) node.input = cleanInputOutput(n.data.input, 'input:');
      if (isSink && n.data.output) node.output = cleanInputOutput(n.data.output, 'output:');
      return node;
    });
    return {
      name: `pipeline_${idx + 1}`,
      nodes: compNodes,
      edges: compEdges.map(e => ({ from: e.source, to: e.target })),
    };
  });

  exportedJson.value = JSON.stringify(pipelines, null, 2);
}

function sendToServer() {
  // Повторяем экспорт пайплайнов (тот же код, что и в exportPipeline)
  // 1. Строим неориентированный граф
  const nodeMap = {};
  nodes.value.forEach(n => { nodeMap[n.id] = n; });
  const adj = {};
  nodes.value.forEach(n => { adj[n.id] = []; });
  edges.value.forEach(e => {
    if (adj[e.source] && !adj[e.source].includes(e.target)) adj[e.source].push(e.target);
    if (adj[e.target] && !adj[e.target].includes(e.source)) adj[e.target].push(e.source);
  });
  // 2. BFS/DFS для поиска компонент связности
  const visited = new Set();
  const components = [];
  for (const n of nodes.value) {
    if (!visited.has(n.id)) {
      const queue = [n.id];
      const compNodes = [];
      while (queue.length) {
        const curr = queue.pop();
        if (visited.has(curr)) continue;
        visited.add(curr);
        compNodes.push(curr);
        for (const nb of adj[curr]) {
          if (!visited.has(nb)) queue.push(nb);
        }
      }
      components.push(compNodes);
    }
  }
  // 3. Для каждой компоненты собираем nodes и edges
  const pipelines = components.map((comp, idx) => {
    const compEdges = edges.value.filter(e => comp.includes(e.source) && comp.includes(e.target));
    const compNodes = comp.map(id => {
      const n = nodeMap[id];
      const isSink = !compEdges.some(e => e.source === id);
      const node = {
        id: n.id,
        type: isSink ? 'sink' : 'processor',
        config: cleanConfig(n.data.config),
      };
      if (!compEdges.some(e => e.target === n.id) && n.data.input) node.input = cleanInputOutput(n.data.input, 'input:');
      if (isSink && n.data.output) node.output = cleanInputOutput(n.data.output, 'output:');
      return node;
    });
    return {
      name: `pipeline_${idx + 1}`,
      nodes: compNodes,
      edges: compEdges.map(e => ({ from: e.source, to: e.target })),
    };
  });
  // Отправляем на сервер (замени URL на свой реальный)
  axios.post('http://localhost:8080/setConfigs', pipelines)
    .then(() => {
      sendStatus.value = 'success';
      sendMessage.value = 'Пайплайны успешно отправлены на сервер!';
      setTimeout(() => { sendStatus.value = ''; sendMessage.value = ''; }, 4000);
    })
    .catch((err) => {
      sendStatus.value = 'error';
      // Парсим ошибку валидации от сервера
      if (err.response && err.response.data && err.response.data["validation info"]) {
        const info = err.response.data["validation info"];
        // Пример: .../pipeline_1_step1.yaml(3,1) field addresse not recognised\n
        // Ищем имя файла и ошибку
        const match = info.match(/pipeline_\d+_(step\d+)\.yaml\(\d+,\d+\)\s*(.*)/);
        if (match) {
          const nodeId = match[1];
          const errorText = match[2].split("\\n")[0];
          sendMessage.value = `Ошибка в ноде ${nodeId}: ${errorText}`;
          if (!serverErrorNodes.value.includes(nodeId)) serverErrorNodes.value.push(nodeId);
        } else {
          sendMessage.value = 'Ошибка: ' + info;
        }
      } else if (err.response && err.response.data && err.response.data.message) {
        sendMessage.value = 'Ошибка: ' + err.response.data.message;
      } else if (err.message) {
        sendMessage.value = 'Ошибка: ' + err.message;
      } else {
        sendMessage.value = 'Ошибка при отправке на сервер.';
      }
      setTimeout(() => { sendStatus.value = ''; sendMessage.value = ''; }, 4000);
    });
}

function onConnect(params) {
  edges.value.push({
    id: `e${params.source}-${params.target}-${Date.now()}`,
    source: params.source,
    target: params.target,
  });
}

function onEdgeUpdate({ edge, connection }) {
  // Обновляем связь (например, при перетаскивании конца стрелки)
  const idx = edges.value.findIndex(e => e.id === edge.id);
  if (idx !== -1) {
    edges.value[idx] = {
      ...edge,
      source: connection.source,
      target: connection.target,
    };
  }
}

function onEdgeDelete({ edge }) {
  // Удаляем связь
  edges.value = edges.value.filter(e => e.id !== edge.id);
}

function openEditor(id) {
  editorNodeId.value = id;
  editorNode.value = nodes.value.find(n => n.id === id);
  // Если нода была с ошибкой сервера — снимаем подсветку
  const idx = serverErrorNodes.value.indexOf(id);
  if (idx !== -1) serverErrorNodes.value.splice(idx, 1);
}
function closeEditor() {
  editorNodeId.value = null;
  editorNode.value = null;
}
function previewConfig(str) {
  if (!str) return '';
  try {
    const obj = JSON.parse(str);
    return JSON.stringify(obj);
  } catch {
    return str.length > 20 ? str.slice(0, 20) + '...' : str;
  }
}

const isCyclic = computed(() => {
  // Алгоритм поиска цикла в ориентированном графе (DFS)
  const visited = new Set();
  const recStack = new Set();
  const nodeIds = nodes.value.map(n => n.id);
  const adj = {};
  nodeIds.forEach(id => (adj[id] = []));
  edges.value.forEach(e => {
    if (adj[e.source]) adj[e.source].push(e.target);
  });
  function dfs(v) {
    if (!visited.has(v)) {
      visited.add(v);
      recStack.add(v);
      for (const n of adj[v]) {
        if (!visited.has(n) && dfs(n)) return true;
        else if (recStack.has(n)) return true;
      }
    }
    recStack.delete(v);
    return false;
  }
  for (const id of nodeIds) {
    if (dfs(id)) return true;
  }
  return false;
});

// Функция для поиска цикла в компоненте (на вход — ids нод и массив edges этой компоненты)
function hasCycleInComponent(nodeIds, compEdges) {
  const adj = {};
  nodeIds.forEach(id => (adj[id] = []));
  compEdges.forEach(e => {
    adj[e.source].push(e.target);
  });
  const visited = new Set();
  const recStack = new Set();
  function dfs(v) {
    if (!visited.has(v)) {
      visited.add(v);
      recStack.add(v);
      for (const n of adj[v]) {
        if (!visited.has(n) && dfs(n)) return true;
        else if (recStack.has(n)) return true;
      }
    }
    recStack.delete(v);
    return false;
  }
  for (const id of nodeIds) {
    if (dfs(id)) return true;
  }
  return false;
}

watch([nodes, edges], () => {
  // 1. Строим неориентированный граф для компонент
  const adj = {};
  nodes.value.forEach(n => { adj[n.id] = []; });
  edges.value.forEach(e => {
    if (adj[e.source] && !adj[e.source].includes(e.target)) adj[e.source].push(e.target);
    if (adj[e.target] && !adj[e.target].includes(e.source)) adj[e.target].push(e.source);
  });
  // 2. Находим компоненты
  const visited = new Set();
  const components = [];
  for (const n of nodes.value) {
    if (!visited.has(n.id)) {
      const queue = [n.id];
      const compNodes = [];
      while (queue.length) {
        const curr = queue.pop();
        if (visited.has(curr)) continue;
        visited.add(curr);
        compNodes.push(curr);
        for (const nb of adj[curr]) {
          if (!visited.has(nb)) queue.push(nb);
        }
      }
      components.push(compNodes);
    }
  }
  // 3. Для каждой компоненты ищем цикл и красим только её edges
  let changed = false;
  components.forEach(comp => {
    const compEdges = edges.value.filter(e => comp.includes(e.source) && comp.includes(e.target));
    const hasCycle = hasCycleInComponent(comp, compEdges);
    compEdges.forEach(e => {
      const newStyle = hasCycle
        ? { stroke: '#ef4444', strokeWidth: 2.5 }
        : { stroke: '#3b82f6', strokeWidth: 2.5 };
      if (JSON.stringify(e.style) !== JSON.stringify(newStyle)) {
        e.style = newStyle;
        changed = true;
      }
    });
  });
  // Только если реально что-то поменялось, триггерим edges.value
  if (changed) {
    edges.value = [...edges.value];
  }
}, { deep: true });

const invalidYamlNodes = computed(() => {
  const invalid = [];
  for (const n of nodes.value) {
    // Проверяем только непустые поля
    if (n.data.config && !isValidYaml(cleanConfig(n.data.config))) invalid.push(n.id);
    if (isInputNode(n.id) && n.data.input && !isValidYaml(cleanInputOutput(n.data.input, 'input:'))) invalid.push(n.id);
    if (isOutputNode(n.id) && n.data.output && !isValidYaml(cleanInputOutput(n.data.output, 'output:'))) invalid.push(n.id);
  }
  return invalid;
});

function isValidYaml(str) {
  try {
    const parsed = yaml.load(str);
    // Только объект или массив считаем валидным
    if (typeof parsed === 'object' && parsed !== null) return true;
    return false;
  } catch {
    return false;
  }
}

const hasInvalidYaml = computed(() => invalidYamlNodes.value.length > 0);
</script>

<style scoped>
.pipeline-wrapper {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
.toolbar {
  position: absolute;
  top: 24px;
  left: 24px;
  z-index: 10;
  display: flex;
  flex-direction: row;
  gap: 16px;
}
.toolbar button {
  background: #3b82f6;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 8px 18px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.2s;
}
.toolbar button:hover {
  background: #2563eb;
}
.toolbar button.disabled {
  background: #cbd5e1;
  color: #64748b;
  cursor: not-allowed;
}
.pipeline-flow { background: #f3f4f6; width: 100vw; height: 100vh; }
.processor-node {
  background: #fff;
  border: 2px solid #3b82f6;
  border-radius: 12px;
  padding: 14px 16px 16px 16px;
  min-width: 220px;
  box-shadow: 0 4px 16px #0002;
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 20px;
  transition: box-shadow 0.2s, border 0.2s;
}
.processor-node.invalid-yaml {
  border-color: #ef4444;
  box-shadow: 0 0 0 2px #ef444455, 0 4px 16px #ef444433;
}
.processor-node.server-error {
  border-color: #ef4444;
  box-shadow: 0 0 0 2px #ef4444bb, 0 4px 16px #ef444433;
  animation: shake 0.2s 2;
}
@keyframes shake {
  0% { transform: translateX(0); }
  25% { transform: translateX(-3px); }
  50% { transform: translateX(3px); }
  75% { transform: translateX(-2px); }
  100% { transform: translateX(0); }
}
.processor-node:focus-within {
  box-shadow: 0 6px 24px #3b82f655;
  border-color: #2563eb;
}
.node-header {
  font-weight: bold;
  margin-bottom: 6px;
  color: #3b82f6;
  font-size: 1.1rem;
}
.node-config-preview {
  color: #64748b;
  font-size: 0.95rem;
  margin-bottom: 4px;
  min-height: 24px;
}
.config-preview {
  color: #334155;
  background: #e0e7ff;
  border-radius: 4px;
  padding: 1px 4px;
  margin-left: 2px;
}
.config-empty {
  color: #cbd5e1;
  font-style: italic;
}
.node-section textarea,
.processor-node textarea {
  width: 100%;
  min-height: 38px;
  border: 1.5px solid #cbd5e1;
  border-radius: 5px;
  padding: 6px 8px;
  font-size: 0.98rem;
  transition: border 0.2s;
  resize: vertical;
  background: #f8fafc;
}
.node-section textarea:focus,
.processor-node textarea:focus {
  border: 1.5px solid #3b82f6;
  outline: none;
  background: #fff;
}
.vue-flow__handle {
  width: 18px !important;
  height: 18px !important;
  border: 2.5px solid #3b82f6 !important;
  background: #fff !important;
  box-shadow: 0 2px 8px #3b82f655;
  transition: border 0.2s, box-shadow 0.2s;
  z-index: 2;
}
.vue-flow__handle:hover {
  border: 2.5px solid #2563eb !important;
  background: #e0e7ff !important;
  box-shadow: 0 4px 16px #2563eb55;
}
.vue-flow__handle:after {
  content: '';
  display: block;
  width: 8px;
  height: 8px;
  background: #3b82f6;
  border-radius: 50%;
  margin: 3px auto;
}
.modal-backdrop {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: #0007;
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
}
.modal-editor {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 8px 32px #0003;
  padding: 28px 32px 20px 32px;
  min-width: 340px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.modal-editor textarea {
  width: 100%;
  min-height: 48px;
  border: 1.5px solid #cbd5e1;
  border-radius: 5px;
  padding: 6px 8px;
  font-size: 1rem;
  background: #f8fafc;
  transition: border 0.2s;
}
.modal-editor textarea:focus {
  border: 1.5px solid #3b82f6;
  outline: none;
  background: #fff;
}
.modal-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 8px;
}
.modal-actions button {
  background: #3b82f6;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 8px 18px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.2s;
}
.modal-actions button:hover {
  background: #2563eb;
}
.exported-json {
  position: absolute;
  top: 100px;
  left: 24px;
  right: 24px;
  background: #fff;
  border: 1.5px solid #3b82f6;
  border-radius: 6px;
  padding: 16px;
  font-size: 0.95rem;
  color: #334155;
  box-shadow: 0 4px 16px #0002;
  z-index: 10;
  overflow-x: auto;
  max-height: calc(100vh - 200px);
}
.exported-json-view {
  position: absolute;
  top: 100px;
  left: 24px;
  right: 24px;
  background: #fff;
  border: 1.5px solid #3b82f6;
  border-radius: 6px;
  padding: 16px;
  font-size: 0.95rem;
  color: #334155;
  box-shadow: 0 4px 16px #0002;
  z-index: 10;
  overflow-x: auto;
  max-height: calc(100vh - 200px);
}
.exported-json-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}
.close-json {
  background: none;
  border: none;
  color: #3b82f6;
  font-size: 1.5rem;
  font-weight: bold;
  cursor: pointer;
  padding: 0 8px;
  border-radius: 4px;
  transition: background 0.2s;
}
.close-json:hover {
  background: #e0e7ff;
}
.cyclic-warning {
  position: absolute;
  top: 70px;
  left: 24px;
  background: #fee2e2;
  color: #b91c1c;
  border: 1.5px solid #f87171;
  border-radius: 6px;
  padding: 10px 18px;
  font-size: 1.05rem;
  z-index: 20;
  box-shadow: 0 2px 8px #f8717133;
}
.invalid-yaml-warning {
  background: #fef2f2;
  color: #b91c1c;
  border: 1.5px solid #ef4444;
  box-shadow: 0 2px 8px #ef444433;
}
.send-status {
  position: absolute;
  top: 70px;
  left: 24px;
  z-index: 20;
  min-width: 320px;
  padding: 12px 22px;
  border-radius: 7px;
  font-size: 1.08rem;
  font-weight: 500;
  box-shadow: 0 2px 12px #0002;
  transition: opacity 0.3s;
}
.send-status.success {
  background: #dcfce7;
  color: #166534;
  border: 1.5px solid #22c55e;
}
.send-status.error {
  background: #fee2e2;
  color: #b91c1c;
  border: 1.5px solid #ef4444;
}
</style>

<style>
/* Глобальные стили для всего приложения */
:global(body), :global(html), :global(#app) {
  margin: 0;
  padding: 0;
  width: 100vw;
  height: 100vh;
  background: #f3f4f6;
  overflow: hidden;
}
</style>
