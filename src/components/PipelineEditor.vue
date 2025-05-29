<template>
  <div class="pipeline-wrapper">
    <div class="toolbar">
      <button @click="addNode">📦 Добавить обработчик</button>
      <button @click="sendToServer"
        :disabled="isCyclic || hasInvalidYaml || hasJoinNodeErrors || hasProcessorNodeErrors || isSending || hasJoinAsOutput"
        :class="{ disabled: isCyclic || hasInvalidYaml || hasJoinNodeErrors || hasProcessorNodeErrors || isSending || hasJoinAsOutput }">
        <span v-if="isSending" class="spinner"></span>
        <span v-else>🚀 Запустить пайплайны</span>
      </button>
      <button @click="clearCanvas" :disabled="isSending" :class="{ disabled: isSending }">🧹 Очистить холст</button>
      <button @click="cleanUpContainers" :disabled="isSending" :class="{ disabled: isSending }">🗑 Очистить контейнеры</button>
      <!-- Кнопка управления конфигами -->
      <div class="config-dropdown-wrapper">
        <button @click="showConfigMenu = !showConfigMenu" title="Управление конфигурациями">
          🗂 Управление конфигурациями
        </button>
        <div v-if="showConfigMenu" class="config-dropdown">
          <button @click="saveConfigToLocal">💾 Сохранить конфиг</button>
          <button @click="loadConfigFromLocal">⭮ Загрузить конфиг</button>
          <button @click="deleteConfigFromLocal">🗑 Удалить конфиг</button>
        </div>
      </div>
    </div>
    <div class="error-stack">
      <div v-if="isCyclic" class="cyclic-warning error-block">
        В графе обнаружен цикл! Экспорт невозможен.
      </div>
      <div v-if="hasInvalidYaml" class="cyclic-warning invalid-yaml-warning error-block">
        В одном или нескольких обработчиках некорректный YAML! Экспорт невозможен.
      </div>
      <div v-if="hasJoinNodeErrors" class="cyclic-warning invalid-yaml-warning error-block">
        В одной или нескольких join-нодах есть ошибки! Экспорт невозможен.
      </div>
      <div v-if="hasJoinAsOutput" class="cyclic-warning error-block">
        Граф не должен заканчиваться join-нодой! Добавьте после join-ноды обычную ноду.
      </div>
      <div v-if="joinOutputError" class="cyclic-warning error-block">
        У одной или нескольких join-нод нет исходящих рёбер.
      </div>
      <!-- Вывод ошибок processor-ноды -->
      <div v-for="(errs, id) in processorNodeErrors" :key="id" class="cyclic-warning invalid-yaml-warning error-block">
        <div v-for="(msg, field) in errs" :key="field">{{ msg }}</div>
      </div>
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
        <div class="processor-node"
          @dblclick="openEditor(id)"
          :class="{
  'invalid-yaml': invalidYamlNodes.includes(id),
  'server-error': serverErrorNodes.includes(id),
  'error-node': errorNodes.includes(id)
}">
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
          <template v-if="isJoinNode(editorNodeId)">
            <label>cacheKey:</label>
            <input v-model="editorNode.data.cacheKey" placeholder="cacheKey (string)" />
            <div v-if="joinCacheKeyError" class="join-error-block">{{ joinCacheKeyError }}</div>
            <label>defaultTTL:</label>
            <input v-model="editorNode.data.defaultTTL" placeholder="defaultTTL (string, опционально)" />
            <div v-if="joinTtlError" class="join-error-block">{{ joinTtlError }}</div>
            <label>filterCondition:</label>
            <input v-model="editorNode.data.filterCondition" placeholder="filterCondition (string, опционально)" />
          </template>
          <template v-else>
            <label>Config (Redpanda Connect):</label>
            <textarea v-model="editorNode.data.config" placeholder="Config (Redpanda Connect)"></textarea>
            <label v-if="isInputNode(editorNodeId)">Input config:</label>
            <textarea v-if="isInputNode(editorNodeId)" v-model="editorNode.data.input" placeholder="Input config"></textarea>
            <label v-if="isOutputNode(editorNodeId)">Output config:</label>
            <textarea v-if="isOutputNode(editorNodeId)" v-model="editorNode.data.output" placeholder="Output config"></textarea>
          </template>
          <div class="modal-actions">
            <button @click="saveEditorNode">Сохранить</button>
          </div>
        </div>
      </div>
    </template>

    <!-- Статус отправки -->
    <div v-if="sendStatus" :class="['send-status', sendStatus === 'success' ? 'success' : 'error']">
      <span>{{ sendMessage }}</span>
      <button v-if="sendStatus === 'error'" class="close-json" @click="() => { sendStatus = ''; sendMessage = ''; }">×</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue';
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
    data: { config: '', input: '', output: '', cacheKey: '', defaultTTL: '', filterCondition: '' },
  },
]);
const edges = ref([]);
let nodeId = 2;
const editorNodeId = ref(null);
const editorNode = ref(null);
const sendStatus = ref('');
const sendMessage = ref('');
const serverErrorNodes = ref([]);
const isSending = ref(false);
const showConfigMenu = ref(false);

function addNode() {
  nodes.value.push({
    id: `step${nodeId++}`,
    type: 'processor',
    position: { x: 100 + 80 * nodeId, y: 100 + 40 * nodeId },
    data: { config: '', input: '', output: '', cacheKey: '', defaultTTL: '', filterCondition: '' },
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
function isJoinNode(id) {
  // Если у ноды больше одного входящего ребра — это join
  return edges.value.filter(e => e.target === id).length > 1;
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

// Проверка: есть ли join-нода без исходящих рёбер (т.е. заканчивается join)
const hasJoinAsOutput = computed(() => {
  return nodes.value.some(n =>
    isJoinNode(n.id) && !edges.value.some(e => e.source === n.id)
  );
});

// Проверка: есть ли join-нода с заполненным output
const joinOutputError = computed(() => {
  return nodes.value.some(n =>
    isJoinNode(n.id) && n.data.output && n.data.output.trim()
  );
});

function sendToServer() {
  isSending.value = true;
  // 1. Строим неориентированный граф
  const nodeMap = {};
  nodes.value.forEach(n => { nodeMap[n.id] = n; });
  const adj = {};
  nodes.value.forEach(n => { adj[n.id] = []; });
  edges.value.forEach(e => {
    if (adj[e.source] && !adj[e.source].includes(e.target)) adj[e.source].push(e.target);
    if (adj[e.target] && !adj[e.target].includes(e.source)) adj[e.target].push(e.source);
  });
  // 2. DFS для поиска компонент связности
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
      const isLast = !compEdges.some(e => e.source === id);
      const isJoin = isJoinNode(id);
      let node = {
        id: n.id,
        type: isJoin ? 'join' : 'processor',
        ...(isJoin ? {} : { config: cleanConfig(n.data.config) }),
      };
      if (isJoin) {
        node.meta = {};
        if (n.data.cacheKey) node.meta.cacheKey = n.data.cacheKey;
        if (n.data.defaultTTL) node.meta.defaultTTL = n.data.defaultTTL;
        if (n.data.filterCondition) node.meta.filterCondition = n.data.filterCondition;
        if (Object.keys(node.meta).length === 0) delete node.meta;
      } else {
        if (!compEdges.some(e => e.target === n.id) && n.data.input) node.input = cleanInputOutput(n.data.input, 'input:');
        if (isLast && n.data.output) node.output = cleanInputOutput(n.data.output, 'output:');
      }
      return node;
    });
    return {
      name: `pipeline_${idx + 1}`,
      nodes: compNodes,
      edges: compEdges.map(e => ({ from: e.source, to: e.target })),
    };
  });
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
      setTimeout(() => { isSending.value = false; }, 400);
      return;
    })
    .finally(() => {
      isSending.value = false;
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

function saveEditorNode() {
  closeEditor();
}

const joinTtlError = ref("");
const joinCacheKeyError = ref("");

watch(
  () => [editorNodeId.value, editorNode.value?.data?.defaultTTL, editorNode.value?.data?.cacheKey],
  ([id, defaultTTL, cacheKey]) => {
    if (id && isJoinNode(id)) {
      // Проверка defaultTTL: строка вида "5m", "10s", "2h" и т.п.
      if (defaultTTL && !/^\d+[smhd]$/.test(defaultTTL.trim())) {
        joinTtlError.value = 'defaultTTL должен быть строкой формата времени, например "5m", "10s", "2h"';
      } else {
        joinTtlError.value = '';
      }
      // Проверка cacheKey: обязательно заполнен
      if (!cacheKey || !cacheKey.trim()) {
        joinCacheKeyError.value = 'cacheKey обязателен';
      } else {
        joinCacheKeyError.value = '';
      }
    } else {
      joinTtlError.value = '';
      joinCacheKeyError.value = '';
    }
  },
  { immediate: true }
);

// 1. Проверка ошибок только для join-ноды
const joinNodeErrors = computed(() => {
  const errors = {};
  for (const n of nodes.value) {
    if (isJoinNode(n.id)) {
      if (n.data.defaultTTL && !/^\d+[smhd]$/.test(n.data.defaultTTL.trim())) {
        errors[n.id] = errors[n.id] || {};
        errors[n.id].defaultTTL = 'defaultTTL должен быть строкой формата времени, например "5m", "10s", "2h"';
      }
      if (!n.data.cacheKey || !n.data.cacheKey.trim()) {
        errors[n.id] = errors[n.id] || {};
        errors[n.id].cacheKey = 'cacheKey обязателен';
      }
    }
  }
  return errors;
});
const hasJoinNodeErrors = computed(() => Object.keys(joinNodeErrors.value).length > 0);

// 2. Проверка обязательности input/output для processor-ноды (НЕ для join)
const processorNodeErrors = computed(() => {
  const errors = {};
  for (const n of nodes.value) {
    if (!isJoinNode(n.id)) {
      if (isInputNode(n.id) && (!n.data.input || !n.data.input.trim())) {
        errors[n.id] = errors[n.id] || {};
        errors[n.id].input = 'Input обязателен для ноды без входящих связей';
      }
      if (isOutputNode(n.id) && (!n.data.output || !n.data.output.trim())) {
        errors[n.id] = errors[n.id] || {};
        errors[n.id].output = 'Output обязателен для ноды без исходящих связей';
      }
    }
  }
  return errors;
});
const hasProcessorNodeErrors = computed(() => Object.keys(processorNodeErrors.value).length > 0);

// 3. Для подсветки ошибок: объединяем joinErrorNodes и processorErrorNodes
const processorErrorNodes = computed(() => Object.keys(processorNodeErrors.value));
const errorNodes = computed(() => {
  // Объединяем все id с ошибками (join + processor)
  return Array.from(new Set([...Object.keys(joinNodeErrors.value), ...Object.keys(processorNodeErrors.value)]));
});

watch(
  () => [nodes.value, edges.value],
  () => {
    // Обновляем состояние ошибок join-ноды при изменении графа
    for (const n of nodes.value) {
      if (isJoinNode(n.id)) {
        // Если у join-ноды есть ошибки, добавляем её в список ошибок сервера
        if (joinNodeErrors.value[n.id]) {
          if (!serverErrorNodes.value.includes(n.id)) {
            serverErrorNodes.value.push(n.id);
          }
        } else {
          // Убираем из ошибок сервера, если ошибки больше нет
          const idx = serverErrorNodes.value.indexOf(n.id);
          if (idx !== -1) {
            serverErrorNodes.value.splice(idx, 1);
          }
        }
      }
    }
  },
  { deep: true }
);

// Миграция для старых нод (добавить недостающие поля)
watch(nodes, (val) => {
  for (const n of val) {
    if (!('cacheKey' in n.data)) n.data.cacheKey = '';
    if (!('defaultTTL' in n.data)) n.data.defaultTTL = '';
    if (!('filterCondition' in n.data)) n.data.filterCondition = '';
  }
}, { immediate: true });

// Сохранить nodes и edges в localStorage
function saveConfigToLocal() {
  const data = {
    nodes: nodes.value,
    edges: edges.value,
    nodeId,
  };
  localStorage.setItem('pipeline_config', JSON.stringify(data));
  sendStatus.value = 'success';
  sendMessage.value = 'Конфигурация сохранена!';
  setTimeout(() => { sendStatus.value = ''; sendMessage.value = ''; }, 2000);
  showConfigMenu.value = false;
}

// Загрузить nodes и edges из localStorage
function loadConfigFromLocal() {
  const data = localStorage.getItem('pipeline_config');
  if (data) {
    try {
      const parsed = JSON.parse(data);
      if (Array.isArray(parsed.nodes) && Array.isArray(parsed.edges)) {
        nodes.value = parsed.nodes;
        edges.value = parsed.edges;
        nodeId = parsed.nodeId || (parsed.nodes.length + 1);
        sendStatus.value = 'success';
        sendMessage.value = 'Конфигурация загружена!';
        setTimeout(() => { sendStatus.value = ''; sendMessage.value = ''; }, 2000);
      }
    } catch {
      sendStatus.value = 'error';
      sendMessage.value = 'Ошибка загрузки конфигурации!';
      setTimeout(() => { sendStatus.value = ''; sendMessage.value = ''; }, 2000);
    }
  } else {
    sendStatus.value = 'error';
    sendMessage.value = 'Нет сохранённой конфигурации!';
    setTimeout(() => { sendStatus.value = ''; sendMessage.value = ''; }, 2000);
  }
  showConfigMenu.value = false;
}

// Удалить конфиг из localStorage
function deleteConfigFromLocal() {
  localStorage.removeItem('pipeline_config');
  sendStatus.value = 'success';
  sendMessage.value = 'Конфигурация удалена!';
  setTimeout(() => { sendStatus.value = ''; sendMessage.value = ''; }, 2000);
  showConfigMenu.value = false;
}

function clearCanvas() {
  nodes.value = [
    {
      id: 'step1',
      type: 'processor',
      position: { x: 100, y: 100 },
      data: { config: '', input: '', output: '', cacheKey: '', defaultTTL: '', filterCondition: '' },
    },
  ];
  edges.value = [];
  nodeId = 2;
  serverErrorNodes.value = [];
}

function cleanUpContainers() {
  isSending.value = true;
  axios.delete('http://localhost:8080/cleanUp')
    .then(() => {
      sendStatus.value = 'success';
      sendMessage.value = 'Контейнеры успешно очищены!';
      setTimeout(() => { sendStatus.value = ''; sendMessage.value = ''; }, 3000);
    })
    .catch((err) => {
      sendStatus.value = 'error';
      sendMessage.value = 'Ошибка очистки контейнеров: ' + (err.response?.data?.message || err.message || 'Неизвестная ошибка');
      setTimeout(() => { sendStatus.value = ''; sendMessage.value = ''; }, 4000);
    })
    .finally(() => {
      isSending.value = false;
    });
}
</script>

<style src="@/assets/PipelineEditor.css" scoped></style>
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
