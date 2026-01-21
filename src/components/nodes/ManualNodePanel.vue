<script setup>
import { computed, nextTick, ref } from 'vue';
import BulkOperations from './ManualNodePanel/BulkOperations.vue';
import NodeActions from './ManualNodePanel/NodeActions.vue';
import NodeTable from './ManualNodePanel/NodeTable.vue';

const props = defineProps({
  manualNodes: { type: Array, default: () => [] },
  paginatedManualNodes: Array,
  filteredManualNodes: { type: Array, default: () => [] },
  currentPage: Number,
  totalPages: Number,
  isSorting: Boolean,
  searchTerm: String,
  viewMode: String,
  groups: { type: Array, default: () => [] },
  activeColorFilter: { type: String, default: null }, // New
  itemsPerPage: { type: Number, default: 24 },
});

const emit = defineEmits([
  'add', 'delete', 'edit', 'changePage', 'update:searchTerm', 'update:viewMode',
  'toggleSort', 'markDirty', 'autoSort', 'deduplicate', 'import', 'deleteAll', 'reorder',
  'rename-group', 'delete-group',
  'set-color-filter', 'batch-update-color', 'batch-delete-nodes',
  'update:itemsPerPage' // Added
]);

const isSelectionMode = ref(false);
const selectedNodeIds = ref(new Set());

const toggleSelectionMode = () => {
    isSelectionMode.value = !isSelectionMode.value;
    selectedNodeIds.value.clear();
};

const toggleNodeSelection = (nodeId) => {
    if (selectedNodeIds.value.has(nodeId)) {
        selectedNodeIds.value.delete(nodeId);
    } else {
        selectedNodeIds.value.add(nodeId);
    }
};

const isAllSelected = computed(() => {
    if (paginatedNodes.value.length === 0) return false;
    return paginatedNodes.value.every(node => selectedNodeIds.value.has(node.id));
});

const selectedCount = computed(() => selectedNodeIds.value.size);

const toggleSelectAll = () => {
    if (isAllSelected.value) {
        // Deselect all on current page
        paginatedNodes.value.forEach(node => selectedNodeIds.value.delete(node.id));
    } else {
        // Select all on current page
        paginatedNodes.value.forEach(node => selectedNodeIds.value.add(node.id));
    }
};

const handleBatchColor = (color) => {
    emit('batch-update-color', Array.from(selectedNodeIds.value), color);
    // Maintain selection? Or clear? Usually clear after action.
    selectedNodeIds.value.clear();
    isSelectionMode.value = false;
};

const handleBatchDelete = () => {
    emit('batch-delete-nodes', Array.from(selectedNodeIds.value));
    selectedNodeIds.value.clear();
    isSelectionMode.value = false;
};


// ... (props setup for computed etc.)
// Re-declare computed in context? No, just use existing but injected
// But need to update the Templates where ManualNodeCard is used

const handleRenameGroup = (oldName) => {
    const newName = prompt('请输入新的分组名称:', oldName);
    if (newName && newName !== oldName) {
        emit('rename-group', oldName, newName);
    }
};

const handleDeleteGroup = (groupName) => {
    if (confirm(`确定要解散分组 "${groupName}" 吗？\n组内节点将被移动到"默认"分组。`)) {
        emit('delete-group', groupName);
    }
};

const draggableManualNodes = computed({
  get: () => [...props.manualNodes],
  set: (val) => emit('reorder', val)
});

const searchModel = computed({
  get: () => props.searchTerm,
  set: (val) => emit('update:searchTerm', val)
});

const paginatedNodes = computed(() => props.paginatedManualNodes);

const handleDelete = (id) => emit('delete', id);
const handleEdit = (id) => emit('edit', id);
const handleAdd = () => emit('add');
const handleChangePage = (page) => {
  emit('changePage', page);
};
const handleSetViewMode = (mode) => emit('update:viewMode', mode);
const handleToggleSort = () => {
  emit('toggleSort');

  // 使用 nextTick 等待状态更新完成后重置分页
  nextTick(() => {
    // 如果已经退出排序模式且没有搜索，重置到第一页
    if (!props.isSorting && !props.searchTerm) {
      emit('changePage', 1);
    }
  });
};
const handleSortEnd = () => {
  emit('markDirty');
  // 手动排序完成后重置到第一页
  if (!props.searchTerm) {
    emit('changePage', 1);
  }
};
const handleAutoSort = () => {
  emit('autoSort');
};
const handleDeduplicate = () => {
  emit('deduplicate');
};
const handleImport = () => {
  emit('import');
};
const handleDeleteAll = () => {
  emit('deleteAll');
};
</script>

<template>
  <div>
    <NodeActions
      :manual-nodes-count="manualNodes.length"
      :filtered-nodes-count="filteredManualNodes.length"
      :search-term="searchModel"
      :active-color-filter="activeColorFilter"
      :view-mode="viewMode"
      :is-sorting="isSorting"
      :is-selection-mode="isSelectionMode"
      @update:search-term="searchModel = $event"
      @update:view-mode="handleSetViewMode"
      @set-color-filter="emit('set-color-filter', $event)"
      @add="handleAdd"
      @import="handleImport"
      @auto-sort="handleAutoSort"
      @deduplicate="handleDeduplicate"
      @toggle-sort="handleToggleSort"
      @delete-all="handleDeleteAll"
      @toggle-selection-mode="toggleSelectionMode"
    />

    <BulkOperations
      :is-selection-mode="isSelectionMode"
      :is-all-selected="isAllSelected"
      :selected-count="selectedCount"
      @toggle-select-all="toggleSelectAll"
      @batch-color="handleBatchColor"
      @batch-delete="handleBatchDelete"
      @exit="() => { selectedNodeIds.clear(); isSelectionMode = false; }"
    />

    <NodeTable
      v-model:draggable-manual-nodes="draggableManualNodes"
      :manual-nodes="manualNodes"
      :paginated-nodes="paginatedNodes"
      :filtered-nodes="filteredManualNodes"
      :search-term="searchModel"
      :is-sorting="isSorting"
      :view-mode="viewMode"
      :is-selection-mode="isSelectionMode"
      :selected-node-ids="selectedNodeIds"
      :current-page="props.currentPage"
      :total-pages="props.totalPages"
      :items-per-page="itemsPerPage"
      @update:items-per-page="emit('update:itemsPerPage', $event)"
      @toggle-select="toggleNodeSelection"
      @edit="handleEdit"
      @delete="handleDelete"
      @sort-end="handleSortEnd"
      @change-page="handleChangePage"
    />
  </div>
</template>
