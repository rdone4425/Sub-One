<!--
  ==================== 优选管理标签页 ====================

  功能说明：
  - 显示所有优选配置列表
  - 支持添加新的优选配置
  - 支持编辑现有优选配置
  - 支持删除优选配置
  - 支持启用/禁用优选配置
  - 支持批量操作

  =====================================================
-->

<script setup lang="ts">
import { computed, defineAsyncComponent, onMounted, ref } from 'vue';
import { storeToRefs } from 'pinia';

import ConfirmModal from '../../components/ui/ConfirmModal.vue';
import EmptyState from '../../components/ui/EmptyState.vue';
import Pagination from '../../components/ui/Pagination.vue';
import { useDataStore } from '../../stores/data';
import { useToastStore } from '../../stores/toast';
import type { OptimalConfig } from '../../types/index';

// 异步加载编辑模态框
const OptimizeEditModal = defineAsyncComponent(
    () => import('./components/OptimizeEditModal.vue')
);

const OptimizeCard = defineAsyncComponent(
    () => import('./components/OptimizeCard.vue')
);

// ==================== Props & Emits ====================

const props = defineProps<{
    tabAction?: { action: string; payload?: any } | null;
}>();

const emit = defineEmits<{
    (e: 'action-handled'): void;
}>();

// ==================== Store & Utils ====================

const { showToast } = useToastStore();
const dataStore = useDataStore();
const { optimalConfigs } = storeToRefs(dataStore);

// ==================== State ====================

const itemsPerPage = 6; // 2行3列布局
const currentPage = ref(1);

const filteredConfigs = computed(() => optimalConfigs.value || []);
const totalPages = computed(
    () => Math.ceil(filteredConfigs.value.length / itemsPerPage) || 1
);

const paginatedConfigs = computed(() => {
    const start = (currentPage.value - 1) * itemsPerPage;
    return filteredConfigs.value.slice(start, start + itemsPerPage);
});

const changePage = (page: number) => {
    if (page >= 1 && page <= totalPages.value) currentPage.value = page;
};

// ==================== Modal States ====================

const showEditModal = ref(false);
const isNewOptimal = ref(false);
const editingOptimal = ref<OptimalConfig | null>(null);

const showDeleteModal = ref(false);
const deletingItemId = ref<string | null>(null);

// ==================== Modal Functions ====================

const openNewOptimalModal = () => {
    isNewOptimal.value = true;
    editingOptimal.value = null;
    showEditModal.value = true;
};

const openEditOptimalModal = (config: OptimalConfig) => {
    isNewOptimal.value = false;
    editingOptimal.value = JSON.parse(JSON.stringify(config));
    showEditModal.value = true;
};

const handleModalConfirm = async (config: OptimalConfig) => {
    try {
        if (isNewOptimal.value) {
            // 添加新配置
            await dataStore.addOptimalConfig(config);
            showToast('✅ 优选配置已添加', 'success');
        } else {
            // 更新配置
            await dataStore.updateOptimalConfig(config);
            showToast('✅ 优选配置已更新', 'success');
        }
        showEditModal.value = false;
        editingOptimal.value = null;
    } catch (error) {
        showToast(`❌ 操作失败: ${error instanceof Error ? error.message : '未知错误'}`, 'error');
    }
};

const confirmDelete = (id: string) => {
    deletingItemId.value = id;
    showDeleteModal.value = true;
};

const handleDeleteConfirm = async () => {
    if (!deletingItemId.value) return;

    try {
        await dataStore.deleteOptimalConfig(deletingItemId.value);
        showToast('✅ 优选配置已删除', 'success');
        showDeleteModal.value = false;
        deletingItemId.value = null;

        // 调整分页
        if (paginatedConfigs.value.length === 0 && currentPage.value > 1) {
            currentPage.value--;
        }
    } catch (error) {
        showToast(`❌ 删除失败: ${error instanceof Error ? error.message : '未知错误'}`, 'error');
    }
};

const handleToggleEnable = async (config: OptimalConfig) => {
    try {
        const updated = { ...config, enabled: !config.enabled };
        await dataStore.updateOptimalConfig(updated);
        showToast(
            updated.enabled ? '✅ 配置已启用' : '✅ 配置已禁用',
            'success'
        );
    } catch (error) {
        showToast(`❌ 操作失败: ${error instanceof Error ? error.message : '未知错误'}`, 'error');
    }
};

const handleRefreshOptimal = async (config: OptimalConfig) => {
    if (!config.sourceUrls || config.sourceUrls.length === 0) {
        showToast('❌ 此配置没有设置源 URL', 'error');
        return;
    }

    try {
        showToast('⏳ 正在刷新优选项...', 'info');
        const success = await dataStore.refreshOptimalConfigFromUrls(config.id);
        if (success) {
            showToast('✅ 优选项已刷新', 'success');
        } else {
            showToast('❌ 刷新失败，请检查源 URL', 'error');
        }
    } catch (error) {
        showToast(`❌ 刷新失败: ${error instanceof Error ? error.message : '未知错误'}`, 'error');
    }
};

// ==================== Lifecycle ====================

onMounted(() => {
    // Load optimal configs if needed
});
</script>

<template>
    <div class="space-y-4 p-4">
        <!-- 标题和操作按钮 -->
        <div class="flex items-center justify-between">
            <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
                🎯 优选管理
            </h2>
            <button
                class="btn-modern flex items-center gap-2 px-4 py-2 text-sm font-semibold"
                @click="openNewOptimalModal"
            >
                <span>➕</span>
                <span>新增优选</span>
            </button>
        </div>

        <!-- 统计信息 -->
        <div class="flex gap-4 text-sm text-gray-600 dark:text-gray-400">
            <div>
                <span class="font-semibold text-gray-900 dark:text-white">{{
                    filteredConfigs.length
                }}</span>
                个配置
            </div>
            <div>
                <span class="font-semibold text-gray-900 dark:text-white">{{
                    filteredConfigs.filter((c) => c.enabled).length
                }}</span>
                个启用中
            </div>
        </div>

        <!-- 优选配置卡片网格 -->
        <template v-if="filteredConfigs.length > 0">
            <div class="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3">
                <template v-for="config in paginatedConfigs" :key="config.id">
                    <OptimizeCard
                        :config="config"
                        @edit="openEditOptimalModal"
                        @delete="confirmDelete"
                        @toggle="handleToggleEnable"
                        @refresh="handleRefreshOptimal"
                    />
                </template>
            </div>

            <!-- 分页 -->
            <Pagination
                :current-page="currentPage"
                :total-pages="totalPages"
                @change-page="changePage"
            />
        </template>

        <!-- 空状态 -->
        <EmptyState
            v-else
            icon="🎯"
            title="暂无优选配置"
            description="点击下方按钮添加你的第一个优选域名/IP配置"
        >
            <button
                class="btn-modern px-6 py-2 text-sm font-semibold"
                @click="openNewOptimalModal"
            >
                ➕ 新增优选
            </button>
        </EmptyState>
    </div>

    <!-- 编辑模态框 -->
    <OptimizeEditModal
        :show="showEditModal"
        :config="editingOptimal"
        :is-new="isNewOptimal"
        @update:show="showEditModal = $event"
        @confirm="handleModalConfirm"
    />

    <!-- 删除确认模态框 -->
    <ConfirmModal
        :show="showDeleteModal"
        size="sm"
        @update:show="showDeleteModal = $event"
        @confirm="handleDeleteConfirm"
    >
        <template #title>
            <h3 class="text-lg font-bold text-gray-900 dark:text-white">
                删除优选配置
            </h3>
        </template>
        <template #body>
            <p class="text-sm text-gray-600 dark:text-gray-400">
                确定要删除这个优选配置吗？此操作无法撤销。
            </p>
        </template>
    </ConfirmModal>
</template>
