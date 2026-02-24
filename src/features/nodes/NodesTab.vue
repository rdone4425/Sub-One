<script setup lang="ts">
import { computed, defineAsyncComponent, onMounted, onUnmounted, ref, watch } from 'vue';

import { storeToRefs } from 'pinia';

import draggable from 'vuedraggable';

import ConfirmModal from '../../components/ui/ConfirmModal.vue';
import BaseModal from '../../components/ui/BaseModal.vue';
import EmptyState from '../../components/ui/EmptyState.vue';
import Pagination from '../../components/ui/Pagination.vue';
import { useBatchSelection } from '../../composables/useBatchSelection';
import { useDataStore } from '../../stores/data';
import { useToastStore } from '../../stores/toast';
import type { Node, OptimalConfig } from '../../types/index';
import { createNode, parseImportText } from '../../utils/importer';
import { fetchOptimalNodesPreview } from '../../utils/api';
import ManualNodeCard from './components/ManualNodeCard.vue';
import OptimalNodeCard from './components/OptimalNodeCard.vue';

const props = defineProps<{
    tabAction?: { action: string } | null;
}>();
const emit = defineEmits<{
    (e: 'save-sort'): void;
    (e: 'toggle-sort'): void;
    (e: 'drag-end', evt: unknown): void;
    (e: 'action-handled'): void;
    (e: 'node-deleted', id: string): void; // Keep for profiles cleanup signal? Store handles cleanup now. Remove?
    // Let's keep specific signals if parent listens. But Store handles logic.
    // Actually, parent (DashboardPage) listening to 'node-deleted' handles `removeIdFromProfiles`.
    // DataStore `deleteNode` ALREADY calls `removeIdFromProfiles`.
    // So we don't need to emit 'node-deleted'.
    // We can remove these emits if we confirm DashboardPage doesn't need them for anything else.
    // DashboardPage uses them to clean up profiles. Store does this now.
    // We can safely remove them.
    // We'll keep drag-end/sort emits for now if sorting state is lifted (it is in store config sortStrategy? No, manual sort).
    // Manual sort is local or store?
    // Store has `manualNodes` which we modify. Sorting is just reordering the array.
}>();
// Async Components
const NodeEditModal = defineAsyncComponent(() => import('./components/NodeEditModal.vue'));
const BulkImportModal = defineAsyncComponent(
    () => import('../../components/ui/BulkImportModal.vue')
);
const SubscriptionImportModal = defineAsyncComponent(
    () => import('./components/SubscriptionImportModal.vue')
);

// Utils
const { showToast } = useToastStore();
const dataStore = useDataStore();
const { manualNodes, optimalConfigs } = storeToRefs(dataStore);

// Local State
const searchTerm = ref('');
const isSortingNodes = ref(false); // Local sort state
const hasUnsavedSortChanges = ref(false);

// Pagination Logic
const itemsPerPage = 15;
const currentPage = ref(1);

const filteredNodes = computed(() => {
    if (!searchTerm.value) return manualNodes.value;
    const term = searchTerm.value.toLowerCase();
    return manualNodes.value.filter(
        (node) =>
            (node.name && node.name.toLowerCase().includes(term)) ||
            (node.server && node.server.toLowerCase().includes(term)) ||
            (node.type && node.type.toLowerCase().includes(term))
    );
});

const totalPages = computed(() => Math.ceil(filteredNodes.value.length / itemsPerPage) || 1);

const paginatedNodes = computed(() => {
    if (isSortingNodes.value) return filteredNodes.value; // Show all when sorting
    const start = (currentPage.value - 1) * itemsPerPage;
    return filteredNodes.value.slice(start, start + itemsPerPage);
});

const changePage = (page: number) => {
    if (page >= 1 && page <= totalPages.value) {
        currentPage.value = page;
    }
};

// 计算优选配置的使用情况
const optimalConfigStats = computed(() => {
    return optimalConfigs.value.map((config) => ({
        ...config,
        usageCount: manualNodes.value.filter(
            (node) => node.optimalConfigIds && node.optimalConfigIds.includes(config.id)
        ).length
    }));
});

// ==================== 优选节点加载 ====================

interface ExpandedVariant {
    expandedServer: string;
    configName: string;
    isGlobal: boolean;
}
interface NodeGroup {
    originalId: string;
    originalName: string;
    protocol: string;
    originalServer: string;
    originalPort: string | number;
    variants: ExpandedVariant[];
}

const expandedGroups = ref<NodeGroup[]>([]);
const expandedLoading = ref(false);

// 展开的扁平列表：每个 variant 单独一条
const flatExpandedNodes = computed(() => {
    const result: Array<ExpandedVariant & { originalName: string; protocol: string; originalPort: string | number }> = [];
    for (const group of expandedGroups.value) {
        for (const v of group.variants) {
            result.push({
                ...v,
                originalName: group.originalName,
                protocol: group.protocol,
                originalPort: group.originalPort
            });
        }
    }
    return result;
});

const filteredExpandedNodes = computed(() => {
    if (!searchTerm.value) return flatExpandedNodes.value;
    const term = searchTerm.value.toLowerCase();
    return flatExpandedNodes.value.filter(
        (n) =>
            n.originalName.toLowerCase().includes(term) ||
            n.expandedServer.toLowerCase().includes(term) ||
            n.configName.toLowerCase().includes(term) ||
            n.protocol.toLowerCase().includes(term)
    );
});

const loadExpandedNodes = async () => {
    expandedLoading.value = true;
    try {
        const result = await fetchOptimalNodesPreview();
        if (result.success && result.groups) {
            expandedGroups.value = result.groups;
        } else {
            expandedGroups.value = [];
        }
    } catch {
        expandedGroups.value = [];
    } finally {
        expandedLoading.value = false;
    }
};

// State
const showNodesMoreMenu = ref(false);
const nodesMoreMenuRef = ref<HTMLElement | null>(null);

// Modal States
const isNewNode = ref(false);
const editingNode = ref<Node | null>(null);
const showNodeModal = ref(false);
const showOptimalConfigsModal = ref(false);
const showBulkImportModal = ref(false);
const showSubscriptionImportModal = ref(false);
const showDeleteNodesModal = ref(false);
const showDeleteSingleNodeModal = ref(false);
const deletingItemId = ref<string | null>(null);

// Batch Select
const {
    isBatchDeleteMode,
    selectedCount,
    toggleBatchDeleteMode,
    isSelected,
    toggleSelection,
    selectAll,
    deselectAll,
    invertSelection,
    getSelectedIds
} = useBatchSelection(paginatedNodes);

// Computed for v-models
const localManualNodes = computed({
    get: () => filteredNodes.value, // For draggable: needs full list or current view? Draggable usually works on full list or visible list in Sort mode.
    // In sort mode, paginatedNodes returns filteredNodes (all).
    // So we can use paginatedNodes.
    set: (value) => {
        // If sorting, we are reordering. We need to apply this reorder to the store.
        // But `filteredNodes` is derived.
        // If we reorder, we update the store.
        // Simpler: Use `manualNodes` directly if no filter?
        // If filtered, sorting is tricky. Usually we disable sort if filtered.
        if (!searchTerm.value) {
            dataStore.manualNodes = value;
            hasUnsavedSortChanges.value = true;
        }
    }
});

// Watch for Cross-Tab Actions
watch(
    () => props.tabAction,
    (val) => {
        if (val && val.action === 'add') {
            handleAddNode();
            emit('action-handled');
        }
    },
    { immediate: true }
);

// Handlers

const handleAddNode = () => {
    isNewNode.value = true;
    editingNode.value = createNode('');
    showNodeModal.value = true;
};

const handleEditNode = (nodeId: string) => {
    const node = manualNodes.value.find((n) => n.id === nodeId);
    if (node) {
        isNewNode.value = false;
        editingNode.value = { ...node };
        showNodeModal.value = true;
    }
};

const handleSaveNode = async (updatedNode?: Node) => {
    const nodeToSave = updatedNode || editingNode.value;
    if (!nodeToSave?.url) return showToast('⚠️ 节点链接不能为空', 'error');

    if (isNewNode.value) {
        dataStore.addNode(nodeToSave);
        currentPage.value = 1; // 优化：新增时跳转到第一页
    } else {
        dataStore.updateNode(nodeToSave);
    }

    await dataStore.saveData('节点');
    showNodeModal.value = false;
};

const handleDeleteNode = (nodeId: string) => {
    deletingItemId.value = nodeId;
    showDeleteSingleNodeModal.value = true;
};

const handleConfirmDeleteSingleNode = async () => {
    if (!deletingItemId.value) return;
    dataStore.deleteNode(deletingItemId.value);

    await dataStore.saveData('节点删除');
    showDeleteSingleNodeModal.value = false;
};

const handleDeleteAllNodes = async () => {
    dataStore.deleteAllNodes();

    await dataStore.saveData('节点清空');
    showDeleteNodesModal.value = false;
};

const handleBatchDelete = async (ids: string[]) => {
    if (!ids || ids.length === 0) return;
    dataStore.batchDeleteNodes(ids);

    await dataStore.saveData(`批量删除 ${ids.length} 个节点`);
    toggleBatchDeleteMode();
    deselectAll(); // Clear selection
};

const handleBulkImport = async (importText: string) => {
    const { subs, nodes } = parseImportText(importText);
    if (subs.length > 0) await dataStore.addSubscriptionsFromBulk(subs);
    if (nodes.length > 0) {
        dataStore.addNodesFromBulk(nodes);
        currentPage.value = 1;
    }

    await dataStore.saveData('批量导入');
    showToast(`✅ 成功导入 ${subs.length} 条订阅和 ${nodes.length} 个手动节点`, 'success');
    showBulkImportModal.value = false;
};

const handleDeduplicate = async () => {
    dataStore.deduplicateNodes();
    await dataStore.saveData('节点去重');
};

const handleAutoSort = async () => {
    dataStore.autoSortNodes();
    await dataStore.saveData('节点排序');
};

const handleToggleSort = () => {
    isSortingNodes.value = !isSortingNodes.value;
    if (!isSortingNodes.value) hasUnsavedSortChanges.value = false;
};

const handleSaveSort = async () => {
    await dataStore.saveData('节点排序');
    hasUnsavedSortChanges.value = false;
    isSortingNodes.value = false;
};

const handleSubscriptionImportSuccess = async () => {
    await dataStore.saveData('导入节点');
    currentPage.value = 1;
};

// UI Handlers
const handleToggleBatchDeleteMode = () => {
    toggleBatchDeleteMode();
    showNodesMoreMenu.value = false;
};

const deleteSelected = () => {
    if (selectedCount.value === 0) return;
    const idsToDelete = getSelectedIds();
    handleBatchDelete(idsToDelete);
};

const handleClickOutside = (event: Event) => {
    if (
        showNodesMoreMenu.value &&
        nodesMoreMenuRef.value &&
        !nodesMoreMenuRef.value.contains(event.target as globalThis.Node)
    ) {
        showNodesMoreMenu.value = false;
    }
};

const handleDragEnd = () => {
    hasUnsavedSortChanges.value = true;
};

onMounted(() => {
    document.addEventListener('click', handleClickOutside);
    loadExpandedNodes();
});
onUnmounted(() => {
    document.removeEventListener('click', handleClickOutside);
});
</script>

<template>
    <div class="w-full">
        <!-- 优选配置统计卡片 -->
        <div v-if="optimalConfigStats.length > 0" class="mb-6 rounded-2xl border border-blue-200 bg-gradient-to-r from-blue-50 to-indigo-50 p-4 dark:border-blue-800 dark:from-blue-900/30 dark:to-indigo-900/30">
            <div class="flex flex-col gap-3">
                <div class="flex items-center justify-between">
                    <h3 class="text-sm font-bold text-blue-900 dark:text-blue-200">
                        🎯 优选配置
                    </h3>
                    <button
                        class="text-xs font-semibold text-blue-600 hover:text-blue-700 dark:text-blue-400 dark:hover:text-blue-300"
                        @click="showOptimalConfigsModal = true"
                    >
                        查看全部 →
                    </button>
                </div>
                <div class="flex flex-wrap gap-2">
                    <div
                        v-for="config in optimalConfigStats"
                        :key="config.id"
                        class="inline-flex items-center gap-2 rounded-lg bg-blue-100 px-3 py-1 text-xs font-semibold text-blue-700 dark:bg-blue-900/50 dark:text-blue-300"
                    >
                        <span>{{ config.name }}</span>
                        <span class="rounded-full bg-blue-200 px-2 py-0.5 text-xs dark:bg-blue-800">
                            {{ config.usageCount }}
                        </span>
                    </div>
                </div>
            </div>
        </div>

        <div class="mb-8 flex flex-col gap-6 sm:flex-row sm:items-center sm:justify-between">
            <div class="flex items-center gap-4"></div>

            <div class="flex w-full flex-wrap items-center gap-2">
                <!-- 搜索框 -->
                <div class="relative mb-2 w-full shrink-0 sm:mb-0 sm:w-56">
                    <input
                        v-model="searchTerm"
                        type="text"
                        placeholder="搜索节点..."
                        class="search-input-unified w-full text-base"
                    />
                    <svg
                        class="absolute left-4 top-1/2 h-5 w-5 -translate-y-1/2 text-gray-400"
                        xmlns="http://www.w3.org/2000/svg"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                    >
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
                        />
                    </svg>
                </div>

                <div class="ml-auto flex flex-wrap items-center gap-2">
                    <!-- 主要操作按钮 -->
                    <div class="flex flex-wrap items-center gap-2">
                        <button
                            class="btn-modern-enhanced btn-add transform px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 sm:px-5 sm:py-2.5 sm:text-sm"
                            @click="handleAddNode"
                        >
                            新增
                        </button>

                        <button
                            class="btn-modern-enhanced btn-import transform px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 sm:px-5 sm:py-2.5 sm:text-sm"
                            @click="showSubscriptionImportModal = true"
                        >
                            导入节点
                        </button>

                        <button
                            v-if="isSortingNodes && hasUnsavedSortChanges"
                            class="btn-modern-enhanced btn-primary flex transform items-center gap-1 px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 sm:gap-2 sm:px-5 sm:py-2.5 sm:text-sm"
                            @click="handleSaveSort"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                class="h-4 w-4 sm:h-5 sm:w-5"
                                fill="none"
                                viewBox="0 0 24 24"
                                stroke="currentColor"
                            >
                                <path
                                    stroke-linecap="round"
                                    stroke-linejoin="round"
                                    stroke-width="2"
                                    d="M5 5a2 2 0 012-2h10a2 2 0 012 2v16l-7-3.5L5 21V5z"
                                />
                            </svg>
                            <span class="hidden sm:inline">保存排序</span>
                        </button>
                        <button
                            :class="
                                isSortingNodes
                                    ? 'btn-modern-enhanced btn-sort sorting flex transform items-center gap-1 px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 sm:gap-2 sm:px-5 sm:py-2.5 sm:text-sm'
                                    : 'btn-modern-enhanced btn-sort flex transform items-center gap-1 px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 sm:gap-2 sm:px-5 sm:py-2.5 sm:text-sm'
                            "
                            @click="handleToggleSort"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                class="h-4 w-4 sm:h-5 sm:w-5"
                                fill="none"
                                viewBox="0 0 24 24"
                                stroke="currentColor"
                            >
                                <path
                                    stroke-linecap="round"
                                    stroke-linejoin="round"
                                    stroke-width="2"
                                    d="M4 8h16M4 16h16"
                                />
                            </svg>
                            <span class="hidden sm:inline">{{
                                isSortingNodes ? '排序中' : '手动排序'
                            }}</span>
                            <span class="sm:hidden">{{ isSortingNodes ? '排序' : '排序' }}</span>
                        </button>
                    </div>

                    <!-- 更多菜单 -->
                    <div ref="nodesMoreMenuRef" class="relative">
                        <button
                            class="hover-lift rounded-2xl p-2 transition-colors hover:bg-gray-200 dark:hover:bg-gray-700 sm:p-4"
                            @click="showNodesMoreMenu = !showNodesMoreMenu"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                class="h-5 w-5 text-gray-600 dark:text-gray-300 sm:h-6 sm:w-6"
                                viewBox="0 0 20 20"
                                fill="currentColor"
                            >
                                <path
                                    d="M6 10a2 2 0 11-4 0 2 2 0 014 0zM12 10a2 2 0 11-4 0 2 2 0 014 0zM16 12a2 2 0 100-4 2 2 0 000 4z"
                                />
                            </svg>
                        </button>
                        <Transition name="slide-fade-sm">
                            <div
                                v-if="showNodesMoreMenu"
                                class="absolute right-0 z-50 mt-2 w-40 rounded-2xl border border-gray-300 bg-white shadow-2xl ring-2 ring-gray-200 dark:border-gray-700 dark:bg-gray-800 dark:ring-gray-700"
                            >
                                <button
                                    class="w-full px-5 py-3 text-left text-base text-gray-700 transition-colors hover:text-gray-900 dark:text-gray-200 dark:hover:text-white"
                                    @click="
                                        handleAutoSort();
                                        showNodesMoreMenu = false;
                                    "
                                >
                                    一键排序
                                </button>
                                <button
                                    class="w-full px-5 py-3 text-left text-base text-gray-700 transition-colors hover:text-gray-900 dark:text-gray-200 dark:hover:text-white"
                                    @click="
                                        handleDeduplicate();
                                        showNodesMoreMenu = false;
                                    "
                                >
                                    一键去重
                                </button>
                                <button
                                    class="w-full px-5 py-3 text-left text-base text-gray-700 transition-colors hover:text-gray-900 dark:text-gray-200 dark:hover:text-white"
                                    @click="handleToggleBatchDeleteMode"
                                >
                                    批量删除
                                </button>
                                <div
                                    class="my-1 border-t border-gray-300 dark:border-gray-700"
                                ></div>
                                <button
                                    class="w-full px-5 py-3 text-left text-base text-red-500 transition-colors hover:text-red-600 dark:hover:text-red-400"
                                    @click="
                                        showDeleteNodesModal = true;
                                        showNodesMoreMenu = false;
                                    "
                                >
                                    清空所有
                                </button>
                            </div>
                        </Transition>
                    </div>
                </div>
            </div>
        </div>

        <!-- 批量操作工具栏 -->
        <Transition name="slide-fade">
            <div
                v-if="isBatchDeleteMode"
                class="mb-6 rounded-2xl border-2 border-emerald-200 bg-linear-to-r from-emerald-50 to-green-50 p-4 shadow-lg dark:border-emerald-800 dark:from-emerald-900/20 dark:to-green-900/20"
            >
                <div class="flex flex-col items-center justify-between gap-4 sm:flex-row">
                    <div
                        class="flex items-center gap-2 text-sm font-semibold text-emerald-700 dark:text-emerald-300"
                    >
                        <svg
                            xmlns="http://www.w3.org/2000/svg"
                            class="h-5 w-5"
                            viewBox="0 0 20 20"
                            fill="currentColor"
                        >
                            <path d="M9 2a1 1 0 000 2h2a1 1 0 100-2H9z" />
                            <path
                                fill-rule="evenodd"
                                d="M4 5a2 2 0 012-2 3 3 0 003 3h2a3 3 0 003-3 2 2 0 012 2v11a2 2 0 01-2 2H6a2 2 0 01-2-2V5zm3 4a1 1 0 000 2h.01a1 1 0 100-2H7zm3 0a1 1 0 000 2h3a1 1 0 100-2h-3zm-3 4a1 1 0 100 2h.01a1 1 0 100-2H7zm3 0a1 1 0 100 2h3a1 1 0 100-2h-3z"
                                clip-rule="evenodd"
                            />
                        </svg>
                        批量删除模式
                        <span
                            v-if="selectedCount > 0"
                            class="ml-2 rounded-full bg-emerald-600 px-3 py-1 text-xs font-bold text-white shadow-md"
                        >
                            已选 {{ selectedCount }}
                        </span>
                    </div>
                    <div class="flex flex-wrap items-center gap-2">
                        <button
                            class="btn-modern-enhanced btn-secondary transform px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 sm:px-4 sm:py-2 sm:text-sm"
                            @click="selectAll"
                        >
                            全选
                        </button>
                        <button
                            class="btn-modern-enhanced btn-secondary transform px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 sm:px-4 sm:py-2 sm:text-sm"
                            @click="invertSelection"
                        >
                            反选
                        </button>
                        <button
                            class="btn-modern-enhanced btn-secondary transform px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 sm:px-4 sm:py-2 sm:text-sm"
                            @click="deselectAll"
                        >
                            清空选择
                        </button>
                        <button
                            :disabled="selectedCount === 0"
                            class="btn-modern-enhanced btn-danger flex transform items-center gap-1 px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 disabled:transform-none disabled:cursor-not-allowed disabled:opacity-50 sm:px-4 sm:py-2 sm:text-sm"
                            @click="deleteSelected"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                class="h-4 w-4"
                                viewBox="0 0 20 20"
                                fill="currentColor"
                            >
                                <path
                                    fill-rule="evenodd"
                                    d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z"
                                    clip-rule="evenodd"
                                />
                            </svg>
                            删除选中 ({{ selectedCount }})
                        </button>
                        <button
                            class="btn-modern-enhanced btn-cancel transform px-3 py-1.5 text-xs font-semibold transition-all duration-300 hover:scale-105 sm:px-4 sm:py-2 sm:text-sm"
                            @click="handleToggleBatchDeleteMode"
                        >
                            取消
                        </button>
                    </div>
                </div>
            </div>
        </Transition>

        <!-- 节点内容区域 -->
        <div v-if="manualNodes.length > 0">
            <draggable
                v-if="isSortingNodes"
                v-model="localManualNodes"
                tag="div"
                class="grid grid-cols-1 gap-4 sm:gap-6 md:grid-cols-2 lg:grid-cols-3 lg:gap-8"
                :item-key="(item: Node) => item.id"
                animation="300"
                :delay="200"
                :delay-on-touch-only="true"
                @end="handleDragEnd"
            >
                <template #item="{ element: node }">
                    <div class="cursor-move">
                        <ManualNodeCard
                            :node="node"
                            :is-batch-mode="isBatchDeleteMode"
                            :is-selected="isSelected(node.id)"
                            :optimal-configs="optimalConfigs"
                            :optimal-config-stats="optimalConfigStats"
                            @edit="handleEditNode(node.id)"
                            @delete="handleDeleteNode(node.id)"
                            @toggle-select="toggleSelection(node.id)"
                        />
                    </div>
                </template>
            </draggable>
            <div
                v-else
                class="grid grid-cols-1 gap-4 sm:gap-6 md:grid-cols-2 lg:grid-cols-3 lg:gap-8"
            >
                <div v-for="node in paginatedNodes" :key="node.id">
                    <ManualNodeCard
                        :node="node"
                        :is-batch-mode="isBatchDeleteMode"
                        :is-selected="isSelected(node.id)"
                        :optimal-configs="optimalConfigs"
                        :optimal-config-stats="optimalConfigStats"
                        @edit="handleEditNode(node.id)"
                        @delete="handleDeleteNode(node.id)"
                        @toggle-select="toggleSelection(node.id)"
                    />
                </div>
            </div>

            <Pagination
                v-if="!isSortingNodes"
                :current-page="currentPage"
                :total-pages="totalPages"
                @change-page="changePage"
            />
        </div>
        <EmptyState
            v-else
            title="没有手动节点"
            description="添加分享链接或单个节点。"
            bg-gradient-class="bg-linear-to-br from-green-500/20 to-emerald-500/20"
            icon-color-class="text-green-500"
        >
            <template #icon>
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    class="h-12 w-12 text-green-500"
                    fill="none"
                    viewBox="0 0 24 24"
                    stroke="currentColor"
                    stroke-width="1.5"
                >
                    <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        d="M10 20l4-16m4 4l-4 4-4-4M6 16l-4-4 4-4"
                    />
                </svg>
            </template>
        </EmptyState>

        <!-- ==================== 优选节点区块 ==================== -->
        <div v-if="flatExpandedNodes.length > 0 || expandedLoading" class="mt-8">
            <!-- 区块标题 -->
            <div class="mb-4 flex items-center justify-between">
                <div class="flex items-center gap-3">
                    <h3 class="text-base font-bold text-gray-700 dark:text-gray-200">
                        🎯 优选节点
                    </h3>
                    <span class="rounded-full bg-amber-100 px-2.5 py-0.5 text-xs font-bold text-amber-700 dark:bg-amber-900/30 dark:text-amber-300">
                        {{ filteredExpandedNodes.length }}
                    </span>
                </div>
                <button
                    class="flex items-center gap-1.5 rounded-lg bg-gray-100 px-3 py-1.5 text-xs font-semibold text-gray-600 transition hover:bg-gray-200 dark:bg-gray-700 dark:text-gray-300 dark:hover:bg-gray-600"
                    :disabled="expandedLoading"
                    @click="loadExpandedNodes"
                >
                    <span :class="{ 'animate-spin': expandedLoading }">🔄</span>
                    刷新
                </button>
            </div>

            <!-- 加载中 -->
            <div v-if="expandedLoading" class="flex items-center justify-center py-10">
                <div class="h-6 w-6 animate-spin rounded-full border-2 border-gray-200 border-t-amber-500"></div>
                <span class="ml-3 text-sm text-gray-500 dark:text-gray-400">正在拉取优选节点...</span>
            </div>

            <!-- 优选节点卡片网格 -->
            <div
                v-else-if="filteredExpandedNodes.length > 0"
                class="grid grid-cols-1 gap-4 sm:gap-6 md:grid-cols-2 lg:grid-cols-3 lg:gap-8"
            >
                <OptimalNodeCard
                    v-for="(node, idx) in filteredExpandedNodes"
                    :key="`opt-${idx}`"
                    :expanded-server="node.expandedServer"
                    :original-port="node.originalPort"
                    :original-name="node.originalName"
                    :protocol="node.protocol"
                    :config-name="node.configName"
                    :is-global="node.isGlobal"
                />
            </div>

            <!-- 搜索无结果时隐藏此区块 -->
        </div>

        <!-- Modals -->
        <BulkImportModal v-model:show="showBulkImportModal" @import="handleBulkImport" />

        <ConfirmModal
            v-model:show="showDeleteNodesModal"
            title="确认清空节点"
            message="您确定要删除所有<strong>手动节点</strong>吗？此操作将标记为待保存，不会影响订阅。"
            type="danger"
            @confirm="handleDeleteAllNodes"
        />

        <ConfirmModal
            v-model:show="showDeleteSingleNodeModal"
            title="确认删除节点"
            message="您确定要删除此手动节点吗？此操作将标记为待保存，不会影响订阅。"
            type="danger"
            @confirm="handleConfirmDeleteSingleNode"
        />

        <NodeEditModal
            v-if="showNodeModal"
            v-model:show="showNodeModal"
            :node="editingNode"
            :is-new="isNewNode"
            @save="handleSaveNode"
        />

        <SubscriptionImportModal
            v-model:show="showSubscriptionImportModal"
            :add-nodes-from-bulk="dataStore.addNodesFromBulk"
            :on-import-success="handleSubscriptionImportSuccess"
        />

        <!-- 查看所有优选配置的模态框 -->
        <BaseModal
            :show="showOptimalConfigsModal"
            size="2xl"
            @update:show="showOptimalConfigsModal = $event"
        >
            <template #title>
                <h3 class="text-lg font-bold text-gray-900 dark:text-white">
                    🎯 所有优选配置
                </h3>
            </template>

            <template #body>
                <div class="space-y-3">
                    <p v-if="optimalConfigStats.length > 0" class="text-sm text-gray-600 dark:text-gray-400">
                        共 <span class="font-semibold">{{ optimalConfigStats.length }}</span> 个优选配置
                    </p>
                    <div v-if="optimalConfigStats.length > 0" class="max-h-96 space-y-2 overflow-y-auto">
                        <div
                            v-for="config in optimalConfigStats"
                            :key="config.id"
                            class="rounded-lg border border-gray-200 bg-gray-50 p-4 dark:border-gray-700 dark:bg-gray-800"
                        >
                            <div class="flex items-start justify-between gap-3">
                                <div class="flex-1">
                                    <h4 class="font-semibold text-gray-900 dark:text-white">
                                        {{ config.name }}
                                    </h4>
                                    <p v-if="config.description" class="mt-1 text-xs text-gray-500 dark:text-gray-400">
                                        {{ config.description }}
                                    </p>
                                    <div class="mt-2 flex flex-wrap gap-2 text-xs">
                                        <span class="inline-flex items-center gap-1 rounded-lg bg-blue-100 px-2 py-1 text-blue-700 dark:bg-blue-900 dark:text-blue-200">
                                            {{ config.type === 'domain' ? '🌐 域名' : config.type === 'ip' ? '📍 IP' : '🔗 混合' }}
                                        </span>
                                        <span class="inline-flex items-center gap-1 rounded-lg bg-purple-100 px-2 py-1 text-purple-700 dark:bg-purple-900 dark:text-purple-200">
                                            📊 {{ config.items?.length || 0 }} 项
                                        </span>
                                    </div>
                                </div>
                                <div class="flex-shrink-0 text-right">
                                    <div class="rounded-lg bg-green-100 px-3 py-2 text-center dark:bg-green-900/30">
                                        <div class="text-2xl font-bold text-green-700 dark:text-green-300">
                                            {{ config.usageCount }}
                                        </div>
                                        <div class="text-xs text-green-600 dark:text-green-400">
                                            个节点使用
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                    <div v-else class="space-y-3 text-center py-8">
                        <p class="text-sm text-gray-500 dark:text-gray-400">
                            暂无优选配置
                        </p>
                    </div>
                </div>
            </template>
        </BaseModal>
    </div>
</template>
