<!--
  ==================== 优选节点预览页 ====================

  功能说明：
  - 调用后端 /api/optimal_nodes/preview 接口
  - 后端实时拉取 sourceUrls 并应用展开逻辑
  - 将所有手动节点按优选配置展开，显示完整的优选节点列表
  - 按原始节点分组展示每个节点对应的所有优选服务器地址
  - 支持搜索过滤
  - 节点总数统计

  ======================================================
-->

<script setup lang="ts">
import { computed, onMounted, ref } from 'vue';
import { fetchOptimalNodesPreview } from '../../utils/api';

// ==================== 类型 ====================

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

// ==================== 状态 ====================

const searchQuery = ref('');
const isLoading = ref(false);
const loadError = ref<string | null>(null);
const allGroups = ref<NodeGroup[]>([]);
const totalExpanded = ref(0);

// ==================== 加载数据 ====================

const loadPreview = async () => {
    isLoading.value = true;
    loadError.value = null;

    try {
        const result = await fetchOptimalNodesPreview();
        if (result.success && result.groups) {
            allGroups.value = result.groups;
            totalExpanded.value = result.totalExpanded ?? 0;
        } else {
            loadError.value = result.error || '获取数据失败';
            allGroups.value = [];
            totalExpanded.value = 0;
        }
    } catch (e) {
        loadError.value = String(e);
        allGroups.value = [];
        totalExpanded.value = 0;
    } finally {
        isLoading.value = false;
    }
};

onMounted(loadPreview);

// ==================== 搜索过滤 ====================

const filteredGroups = computed((): NodeGroup[] => {
    const q = searchQuery.value.trim().toLowerCase();
    if (!q) return allGroups.value;
    return allGroups.value
        .map((group) => {
            const nameMatch =
                group.originalName.toLowerCase().includes(q) ||
                group.protocol.toLowerCase().includes(q) ||
                group.originalServer.toLowerCase().includes(q);
            if (nameMatch) return group;
            const variantsMatch = group.variants.filter(
                (v) =>
                    v.expandedServer.toLowerCase().includes(q) ||
                    v.configName.toLowerCase().includes(q)
            );
            if (variantsMatch.length > 0) return { ...group, variants: variantsMatch };
            return null;
        })
        .filter((g): g is NodeGroup => g !== null);
});

const filteredExpandedCount = computed(() =>
    filteredGroups.value.reduce((sum, g) => sum + g.variants.length, 0)
);

// ==================== 协议颜色 ====================

const protocolColorMap: Record<string, string> = {
    trojan:      'bg-purple-100 text-purple-700 dark:bg-purple-900/50 dark:text-purple-300',
    vless:       'bg-blue-100 text-blue-700 dark:bg-blue-900/50 dark:text-blue-300',
    vmess:       'bg-indigo-100 text-indigo-700 dark:bg-indigo-900/50 dark:text-indigo-300',
    ss:          'bg-green-100 text-green-700 dark:bg-green-900/50 dark:text-green-300',
    shadowsocks: 'bg-green-100 text-green-700 dark:bg-green-900/50 dark:text-green-300',
    hysteria2:   'bg-rose-100 text-rose-700 dark:bg-rose-900/50 dark:text-rose-300',
    hysteria:    'bg-rose-100 text-rose-700 dark:bg-rose-900/50 dark:text-rose-300',
    tuic:        'bg-orange-100 text-orange-700 dark:bg-orange-900/50 dark:text-orange-300',
    wireguard:   'bg-teal-100 text-teal-700 dark:bg-teal-900/50 dark:text-teal-300',
};

function getProtocolColor(protocol: string): string {
    return (
        protocolColorMap[protocol.toLowerCase()] ??
        'bg-gray-100 text-gray-600 dark:bg-gray-700 dark:text-gray-300'
    );
}
</script>

<template>
    <div class="space-y-4 p-4">
        <!-- 标题和统计 -->
        <div class="flex flex-wrap items-center justify-between gap-3">
            <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
                🚀 优选节点预览
            </h2>
            <div class="flex items-center gap-3">
                <div v-if="!isLoading" class="flex gap-4 text-sm text-gray-600 dark:text-gray-400">
                    <span>
                        <span class="font-semibold text-gray-900 dark:text-white">
                            {{ filteredGroups.length }}
                        </span>
                        个原始节点
                    </span>
                    <span>→</span>
                    <span>
                        <span class="font-semibold text-teal-600 dark:text-teal-400">
                            {{ filteredExpandedCount }}
                        </span>
                        个优选节点
                    </span>
                </div>
                <button
                    class="flex items-center gap-1.5 rounded-lg bg-gray-100 px-3 py-1.5 text-xs font-semibold text-gray-600 transition hover:bg-gray-200 dark:bg-gray-700 dark:text-gray-300 dark:hover:bg-gray-600"
                    :disabled="isLoading"
                    @click="loadPreview"
                >
                    <span :class="{ 'animate-spin': isLoading }">🔄</span>
                    刷新
                </button>
            </div>
        </div>

        <!-- 加载中 -->
        <div v-if="isLoading" class="flex items-center justify-center py-20">
            <div class="h-8 w-8 animate-spin rounded-full border-2 border-gray-200 border-t-indigo-500"></div>
            <span class="ml-3 text-sm text-gray-500 dark:text-gray-400">正在拉取并计算优选节点...</span>
        </div>

        <!-- 加载失败 -->
        <div
            v-else-if="loadError"
            class="flex flex-col items-center justify-center rounded-2xl border-2 border-dashed border-red-200 py-16 dark:border-red-800"
        >
            <span class="mb-3 text-4xl">❌</span>
            <p class="text-base font-semibold text-red-600 dark:text-red-400">加载失败</p>
            <p class="mt-1 text-sm text-gray-500 dark:text-gray-400">{{ loadError }}</p>
            <button
                class="mt-4 rounded-lg bg-indigo-100 px-4 py-2 text-sm font-semibold text-indigo-700 hover:bg-indigo-200 dark:bg-indigo-900/40 dark:text-indigo-300"
                @click="loadPreview"
            >
                重试
            </button>
        </div>

        <!-- 无数据 -->
        <div
            v-else-if="allGroups.length === 0"
            class="flex flex-col items-center justify-center rounded-2xl border-2 border-dashed border-gray-200 py-16 dark:border-gray-700"
        >
            <span class="mb-3 text-4xl">🎯</span>
            <p class="text-base font-semibold text-gray-700 dark:text-gray-300">暂无优选节点</p>
            <p class="mt-1 text-sm text-gray-500 dark:text-gray-400">
                请前往「优选管理」添加优选配置，或前往「手动节点」添加启用的节点
            </p>
        </div>

        <template v-else>
            <!-- 搜索框 -->
            <div class="relative">
                <span class="absolute left-3 top-1/2 -translate-y-1/2 text-gray-400">🔍</span>
                <input
                    v-model="searchQuery"
                    type="text"
                    placeholder="搜索节点名称、服务器地址、配置名..."
                    class="w-full rounded-xl border border-gray-200 bg-white py-2.5 pl-9 pr-4 text-sm text-gray-900 placeholder-gray-400 outline-none transition focus:border-indigo-400 focus:ring-2 focus:ring-indigo-100 dark:border-gray-700 dark:bg-gray-800 dark:text-white dark:placeholder-gray-500 dark:focus:border-indigo-500 dark:focus:ring-indigo-900/30"
                />
            </div>

            <!-- 无搜索结果 -->
            <div
                v-if="filteredGroups.length === 0"
                class="py-12 text-center text-sm text-gray-500 dark:text-gray-400"
            >
                未找到匹配的节点
            </div>

            <!-- 节点分组列表 -->
            <div v-else class="space-y-4">
                <div
                    v-for="group in filteredGroups"
                    :key="group.originalId"
                    class="overflow-hidden rounded-2xl border border-gray-200 bg-white shadow-sm dark:border-gray-700 dark:bg-gray-800"
                >
                    <!-- 原始节点信息头部 -->
                    <div
                        class="flex flex-wrap items-center gap-3 border-b border-gray-100 bg-gray-50 px-4 py-3 dark:border-gray-700 dark:bg-gray-700/50"
                    >
                        <span
                            class="inline-flex items-center rounded-lg px-2 py-0.5 text-xs font-bold"
                            :class="getProtocolColor(group.protocol)"
                        >
                            {{ group.protocol.toUpperCase() }}
                        </span>
                        <span class="font-semibold text-gray-900 dark:text-white">
                            {{ group.originalName }}
                        </span>
                        <span
                            v-if="group.originalServer"
                            class="font-mono text-xs text-gray-400 dark:text-gray-500"
                        >
                            {{ group.originalServer }}{{ group.originalPort ? ':' + group.originalPort : '' }}
                        </span>
                        <span class="ml-auto text-xs text-gray-400 dark:text-gray-500">
                            → {{ group.variants.length }} 个优选节点
                        </span>
                    </div>

                    <!-- 展开后的优选节点列表 -->
                    <div class="grid grid-cols-1 gap-px bg-gray-100 dark:bg-gray-700 sm:grid-cols-2 lg:grid-cols-3">
                        <div
                            v-for="(variant, idx) in group.variants"
                            :key="idx"
                            class="flex items-center justify-between gap-2 bg-white px-4 py-2.5 dark:bg-gray-800"
                        >
                            <div class="min-w-0 flex-1">
                                <p
                                    class="truncate font-mono text-sm font-medium text-gray-800 dark:text-gray-100"
                                    :title="variant.expandedServer"
                                >
                                    {{ variant.expandedServer }}
                                </p>
                                <p class="mt-0.5 truncate text-xs text-gray-400 dark:text-gray-500">
                                    <span
                                        v-if="variant.isGlobal"
                                        class="mr-1 rounded bg-green-100 px-1 text-green-600 dark:bg-green-900/40 dark:text-green-400"
                                    >全局</span>
                                    {{ variant.configName }}
                                </p>
                            </div>
                            <span class="shrink-0 text-xs text-gray-300 dark:text-gray-600">
                                #{{ idx + 1 }}
                            </span>
                        </div>
                    </div>
                </div>
            </div>
        </template>
    </div>
</template>
