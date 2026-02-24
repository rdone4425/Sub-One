<!--
  ==================== 订阅组编辑模态框 ====================
  
  功能说明：
  - 创建或编辑订阅组（Profile）
  - 选择包含的订阅和手动节点
  - 支持搜索和智能筛选（国家/地区别名匹配）
  - 配置订阅组属性（名称、ID、后端、配置、到期时间）
  - 批量选择/取消选择功能
  
  配置项：
  - 基本信息：订阅组名称、自定义ID
  - 高级设置：自定义后端、自定义配置、到期时间
  - 内容选择：订阅列表、手动节点列表
  
  ==================================================
-->

<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue';

import Modal from '../../../components/ui/BaseModal.vue';
import type { Node, Profile, Subscription } from '../../../types/index';
import { fetchOptimalNodesPreview } from '../../../utils/api';
import { filterNodes } from '../../../utils/search';
import { generateShortId } from '../../../utils/utils';

const props = withDefaults(
    defineProps<{
        show: boolean;
        profile?: Profile | null;
        isNew?: boolean;
        allSubscriptions?: Subscription[];
        allManualNodes?: Node[];
    }>(),
    {
        isNew: false,
        allSubscriptions: () => [],
        allManualNodes: () => []
    }
);

const emit = defineEmits<{
    (e: 'update:show', value: boolean): void;
    (e: 'save', profile: Profile): void;
}>();

const localProfile = ref<Profile>({
    id: '',
    name: '',
    enabled: true,
    subscriptions: [],
    manualNodes: [],
    customId: '',
    expiresAt: '',
    type: 'base64'
});
const subscriptionSearchTerm = ref('');
const nodeSearchTerm = ref('');

const handleGenerateShortId = () => {
    localProfile.value.customId = generateShortId(8);
};

const filteredSubscriptions = computed(() => {
    // 基础过滤：保留已启用的，或者虽然已禁用但当前已被选中的
    let candidates = props.allSubscriptions.filter((sub) => {
        const isEnabled = sub.enabled;
        const isSelected =
            localProfile.value.subscriptions && localProfile.value.subscriptions.includes(sub.id);
        return isEnabled || isSelected;
    });

    return filterNodes(candidates, subscriptionSearchTerm.value);
});

const filteredManualNodes = computed(() => {
    return filterNodes(props.allManualNodes, nodeSearchTerm.value);
});

watch(
    () => props.profile,
    (newProfile) => {
        if (newProfile) {
            const profileCopy = JSON.parse(JSON.stringify(newProfile));
            // Format date for input[type=date]
            if (profileCopy.expiresAt) {
                try {
                    profileCopy.expiresAt = new Date(profileCopy.expiresAt)
                        .toISOString()
                        .split('T')[0];
                } catch (e) {
                    console.error('Error parsing expiresAt date:', e);
                    profileCopy.expiresAt = '';
                }
            }
            localProfile.value = profileCopy;
        } else {
            localProfile.value = {
                id: '',
                name: '',
                enabled: true,
                subscriptions: [],
                manualNodes: [],
                customId: '',
                expiresAt: '',
                type: 'base64'
            };
        }
    },
    { deep: true, immediate: true }
);

const handleConfirm = () => {
    const profileToSave = JSON.parse(JSON.stringify(localProfile.value));
    if (profileToSave.expiresAt) {
        try {
            // Set time to the end of the selected day in local time, then convert to ISO string
            const date = new Date(profileToSave.expiresAt);
            date.setHours(23, 59, 59, 999);
            profileToSave.expiresAt = date.toISOString();
        } catch (e) {
            console.error('Error processing expiresAt date:', e);
            // Decide how to handle error: save as is, or clear it
            profileToSave.expiresAt = '';
        }
    }
    emit('save', profileToSave);
};

const toggleSelection = (listName: 'subscriptions' | 'manualNodes', id: string) => {
    // 确保数组已经初始化
    if (!localProfile.value[listName]) {
        localProfile.value[listName] = [];
    }
    const list = localProfile.value[listName];
    const index = list.indexOf(id);
    if (index > -1) {
        list.splice(index, 1);
    } else {
        list.push(id);
    }
};

const handleSelectAll = (
    listName: 'subscriptions' | 'manualNodes',
    sourceArray: { id: string }[]
) => {
    const currentSelection = new Set(localProfile.value[listName]);
    sourceArray.forEach((item) => currentSelection.add(item.id));
    localProfile.value[listName] = Array.from(currentSelection);
};

const handleDeselectAll = (
    listName: 'subscriptions' | 'manualNodes',
    sourceArray: { id: string }[]
) => {
    const sourceIds = sourceArray.map((item) => item.id);
    // 确保数组已经初始化
    if (!localProfile.value[listName]) {
        localProfile.value[listName] = [];
    }
    localProfile.value[listName] = (localProfile.value[listName] as string[]).filter(
        (id) => !sourceIds.includes(id)
    );
};

// ==================== 优选节点预览 ====================

interface ExpandedVariant {
    expandedServer: string;
    configName: string;
    isGlobal: boolean;
    originalPort: string | number;
    originalName: string;
}

const allOptimalGroups = ref<Array<{
    originalId: string;
    originalName: string;
    originalPort: string | number;
    variants: Array<{ expandedServer: string; configName: string; isGlobal: boolean }>;
}>>([]);
const optimalLoading = ref(false);
const showOptimalPreview = ref(false);

// 当前选中的手动节点所对应的展开列表
const previewExpandedNodes = computed((): ExpandedVariant[] => {
    const selected = new Set(localProfile.value.manualNodes || []);
    const result: ExpandedVariant[] = [];
    for (const group of allOptimalGroups.value) {
        if (!selected.has(group.originalId)) continue;
        for (const v of group.variants) {
            result.push({
                ...v,
                originalPort: group.originalPort,
                originalName: group.originalName
            });
        }
    }
    return result;
});

const loadOptimalPreview = async () => {
    if (allOptimalGroups.value.length > 0) return; // 已加载，不重复请求
    optimalLoading.value = true;
    try {
        const result = await fetchOptimalNodesPreview();
        if (result.success && result.groups) {
            allOptimalGroups.value = result.groups;
        }
    } finally {
        optimalLoading.value = false;
    }
};

// 一键选中所有手动节点
const selectAllManualNodes = () => {
    localProfile.value.manualNodes = props.allManualNodes.map((n) => n.id);
};

onMounted(loadOptimalPreview);
</script>

<template>
    <Modal
        :show="show"
        size="2xl"
        @update:show="emit('update:show', $event)"
        @confirm="handleConfirm"
    >
        <template #title>
            <h3 class="text-xl font-bold text-gray-800 dark:text-white">
                {{ isNew ? '新增订阅组' : '编辑订阅组' }}
            </h3>
        </template>
        <template #body>
            <div v-if="localProfile" class="space-y-6">
                <!-- 表单区域 -->
                <div class="grid grid-cols-1 gap-6 md:grid-cols-2">
                    <!-- 订阅组名称 -->
                    <div>
                        <label
                            for="profile-name"
                            class="mb-2 block text-sm font-bold text-gray-700 dark:text-gray-300"
                        >
                            订阅组名称
                        </label>
                        <input
                            id="profile-name"
                            v-model="localProfile.name"
                            type="text"
                            placeholder="例如：家庭共享"
                            class="input-modern-enhanced"
                        />
                    </div>

                    <!-- 自定义 ID -->
                    <div>
                        <label
                            for="profile-custom-id"
                            class="mb-2 block text-sm font-bold text-gray-700 dark:text-gray-300"
                        >
                            自定义 ID (订阅短链)
                        </label>
                        <div class="flex gap-2">
                            <input
                                id="profile-custom-id"
                                v-model="localProfile.customId"
                                type="text"
                                placeholder="如: home, game (限字母、数字、-、_)"
                                class="input-modern-enhanced flex-1"
                            />
                            <button
                                type="button"
                                class="rounded-xl bg-gray-100 px-3 py-2 text-gray-600 shadow-sm transition-colors hover:bg-gray-200 dark:bg-gray-700 dark:text-gray-300 dark:hover:bg-gray-600"
                                title="生成随机短 ID"
                                @click="handleGenerateShortId"
                            >
                                <svg
                                    xmlns="http://www.w3.org/2000/svg"
                                    class="h-5 w-5"
                                    fill="none"
                                    viewBox="0 0 24 24"
                                    stroke="currentColor"
                                >
                                    <path
                                        stroke-linecap="round"
                                        stroke-linejoin="round"
                                        stroke-width="2"
                                        d="M19.428 15.428a2 2 0 00-1.022-.547l-2.387-.477a6 6 0 00-3.86.517l-.318.158a6 6 0 01-3.86.517L6.05 15.21a2 2 0 00-1.806.547M8 4h8l-1 1v5.172a2 2 0 00.586 1.414l5 5c1.26 1.26.367 3.414-1.415 3.414H4.828c-1.782 0-2.674-2.154-1.414-3.414l5-5A2 2 0 009 10.172V5L8 4z"
                                    />
                                </svg>
                            </button>
                        </div>
                        <p class="mt-1.5 text-xs text-gray-400">
                            设置后，订阅链接会变为：/token/<span
                                class="font-bold text-indigo-500"
                                >{{ localProfile.customId || 'id' }}</span
                            >
                        </p>
                    </div>

                    <!-- 到期时间 -->
                    <div class="md:col-span-1">
                        <label
                            for="profile-expires-at"
                            class="mb-2 block text-sm font-bold text-gray-700 dark:text-gray-300"
                        >
                            到期时间 (可选)
                        </label>
                        <div class="relative">
                            <input
                                id="profile-expires-at"
                                v-model="localProfile.expiresAt"
                                type="date"
                                class="input-modern-enhanced"
                            />
                        </div>
                        <p class="mt-1.5 text-xs text-gray-400">
                            设置此订阅组的到期时间，到期后将返回默认节点。
                        </p>
                    </div>
                </div>

                <!-- 选择区域 -->
                <div class="grid grid-cols-1 gap-6 pt-2 md:grid-cols-2">
                    <!-- 选择订阅 -->
                    <div class="flex h-80 flex-col">
                        <div class="mb-3 flex items-center justify-between">
                            <h4 class="text-sm font-bold text-gray-700 dark:text-gray-300">
                                选择订阅
                            </h4>
                            <div class="space-x-3 text-sm">
                                <button
                                    class="font-medium text-indigo-600 transition-colors hover:text-indigo-700"
                                    @click="handleSelectAll('subscriptions', filteredSubscriptions)"
                                >
                                    全选
                                </button>
                                <button
                                    class="font-medium text-indigo-600 transition-colors hover:text-indigo-700"
                                    @click="
                                        handleDeselectAll('subscriptions', filteredSubscriptions)
                                    "
                                >
                                    全不选
                                </button>
                            </div>
                        </div>

                        <div class="relative mb-3">
                            <input
                                v-model="subscriptionSearchTerm"
                                type="text"
                                placeholder="搜索订阅..."
                                class="search-input-unified"
                            />
                            <svg
                                class="absolute left-3.5 top-1/2 h-4 w-4 -translate-y-1/2 text-gray-400"
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

                        <div
                            class="custom-scrollbar flex-1 overflow-y-auto rounded-xl border border-gray-300 bg-gray-50 p-2 dark:border-gray-700 dark:bg-gray-800"
                        >
                            <div v-if="filteredSubscriptions.length > 0" class="space-y-1">
                                <div v-for="sub in filteredSubscriptions" :key="sub.id">
                                    <label
                                        class="group relative flex cursor-pointer items-center overflow-hidden rounded-lg p-3 transition-all duration-200"
                                        :class="
                                            localProfile.subscriptions?.includes(sub.id)
                                                ? 'bg-indigo-50 dark:bg-indigo-900/20'
                                                : 'hover:bg-gray-50 dark:hover:bg-gray-700/50'
                                        "
                                    >
                                        <div
                                            v-if="localProfile.subscriptions?.includes(sub.id)"
                                            class="absolute bottom-0 left-0 top-0 w-1 bg-linear-to-b from-indigo-500 to-purple-500"
                                        ></div>
                                        <input
                                            type="checkbox"
                                            :checked="localProfile.subscriptions?.includes(sub.id)"
                                            class="mr-3 h-5 w-5 rounded border-gray-300 text-indigo-600 transition-colors"
                                            @change="toggleSelection('subscriptions', sub.id)"
                                        />
                                        <span
                                            class="select-none truncate text-sm font-medium text-gray-700 dark:text-gray-200"
                                        >
                                            {{ sub.name || '未命名订阅' }}
                                            <span
                                                v-if="!sub.enabled"
                                                class="ml-1 text-xs text-red-500"
                                                >(已禁用)</span
                                            >
                                        </span>
                                    </label>
                                </div>
                            </div>
                            <div
                                v-else
                                class="flex h-full flex-col items-center justify-center text-sm text-gray-400"
                            >
                                <svg
                                    xmlns="http://www.w3.org/2000/svg"
                                    class="mb-2 h-8 w-8 opacity-50"
                                    fill="none"
                                    viewBox="0 0 24 24"
                                    stroke="currentColor"
                                >
                                    <path
                                        stroke-linecap="round"
                                        stroke-linejoin="round"
                                        stroke-width="2"
                                        d="M9.172 16.172a4 4 0 015.656 0M9 10h.01M15 10h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
                                    />
                                </svg>
                                没有找到订阅
                            </div>
                        </div>
                    </div>

                    <!-- 选择手动节点 -->
                    <div class="flex h-80 flex-col">
                        <div class="mb-3 flex items-center justify-between">
                            <h4 class="text-sm font-bold text-gray-700 dark:text-gray-300">
                                选择手动节点
                            </h4>
                            <div class="space-x-3 text-sm">
                                <button
                                    class="font-medium text-indigo-600 transition-colors hover:text-indigo-700"
                                    @click="handleSelectAll('manualNodes', filteredManualNodes)"
                                >
                                    全选
                                </button>
                                <button
                                    class="font-medium text-indigo-600 transition-colors hover:text-indigo-700"
                                    @click="handleDeselectAll('manualNodes', filteredManualNodes)"
                                >
                                    全不选
                                </button>
                            </div>
                        </div>

                        <div class="relative mb-3">
                            <input
                                v-model="nodeSearchTerm"
                                type="text"
                                placeholder="搜索节点..."
                                class="search-input-unified"
                            />
                            <svg
                                class="absolute left-3.5 top-1/2 h-4 w-4 -translate-y-1/2 text-gray-400"
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

                        <div
                            class="custom-scrollbar flex-1 overflow-y-auto rounded-xl border border-gray-300 bg-gray-50 p-2 dark:border-gray-700 dark:bg-gray-800"
                        >
                            <div v-if="filteredManualNodes.length > 0" class="space-y-1">
                                <div v-for="node in filteredManualNodes" :key="node.id">
                                    <label
                                        class="group relative flex cursor-pointer items-center overflow-hidden rounded-lg p-3 transition-all duration-200"
                                        :class="
                                            localProfile.manualNodes?.includes(node.id)
                                                ? 'bg-indigo-50 dark:bg-indigo-900/20'
                                                : 'hover:bg-gray-50 dark:hover:bg-gray-700/50'
                                        "
                                    >
                                        <div
                                            v-if="localProfile.manualNodes?.includes(node.id)"
                                            class="absolute bottom-0 left-0 top-0 w-1 bg-linear-to-b from-indigo-500 to-purple-500"
                                        ></div>
                                        <input
                                            type="checkbox"
                                            :checked="localProfile.manualNodes?.includes(node.id)"
                                            class="mr-3 h-5 w-5 rounded border-gray-300 text-indigo-600 transition-colors"
                                            @change="toggleSelection('manualNodes', node.id)"
                                        />
                                        <span
                                            class="select-none truncate text-sm font-medium text-gray-700 dark:text-gray-200"
                                            >{{ node.name || '未命名节点' }}</span
                                        >
                                    </label>
                                </div>
                            </div>
                            <div
                                v-else
                                class="flex h-full flex-col items-center justify-center text-sm text-gray-400"
                            >
                                <svg
                                    xmlns="http://www.w3.org/2000/svg"
                                    class="mb-2 h-8 w-8 opacity-50"
                                    fill="none"
                                    viewBox="0 0 24 24"
                                    stroke="currentColor"
                                >
                                    <path
                                        stroke-linecap="round"
                                        stroke-linejoin="round"
                                        stroke-width="2"
                                        d="M9.172 16.172a4 4 0 015.656 0M9 10h.01M15 10h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
                                    />
                                </svg>
                                没有找到节点
                            </div>
                        </div>
                    </div>
                </div>

                <!-- 🎯 优选节点展开预览 -->
                <div class="rounded-xl border border-amber-200 bg-amber-50 dark:border-amber-800 dark:bg-amber-900/20">
                    <!-- 标题行（可展开/收起） -->
                    <button
                        type="button"
                        class="flex w-full items-center justify-between gap-3 px-4 py-3 text-left"
                        @click="showOptimalPreview = !showOptimalPreview"
                    >
                        <div class="flex items-center gap-2">
                            <span class="text-sm font-bold text-amber-700 dark:text-amber-300">
                                🎯 优选节点预览
                            </span>
                            <span
                                v-if="!optimalLoading"
                                class="rounded-full bg-amber-200 px-2 py-0.5 text-xs font-bold text-amber-800 dark:bg-amber-800 dark:text-amber-200"
                            >
                                {{ previewExpandedNodes.length }} 个
                            </span>
                            <span v-else class="text-xs text-amber-600 dark:text-amber-400">加载中...</span>
                        </div>
                        <div class="flex items-center gap-3">
                            <!-- 一键选择所有手动节点 -->
                            <span
                                class="rounded-lg bg-amber-100 px-3 py-1 text-xs font-semibold text-amber-700 transition hover:bg-amber-200 dark:bg-amber-800 dark:text-amber-200 dark:hover:bg-amber-700"
                                title="选中所有手动节点，确保所有优选节点都包含在订阅组中"
                                @click.stop="selectAllManualNodes"
                            >
                                包含所有手动节点
                            </span>
                            <svg
                                class="h-4 w-4 text-amber-600 transition-transform duration-200 dark:text-amber-400"
                                :class="{ 'rotate-180': showOptimalPreview }"
                                fill="none" viewBox="0 0 24 24" stroke="currentColor"
                            >
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
                            </svg>
                        </div>
                    </button>

                    <!-- 展开的节点列表 -->
                    <div v-if="showOptimalPreview" class="border-t border-amber-200 px-4 pb-4 pt-3 dark:border-amber-800">
                        <!-- 加载中 -->
                        <div v-if="optimalLoading" class="flex items-center gap-2 py-4">
                            <div class="h-4 w-4 animate-spin rounded-full border-2 border-amber-300 border-t-amber-600"></div>
                            <span class="text-xs text-amber-600 dark:text-amber-400">正在拉取优选节点...</span>
                        </div>

                        <!-- 空状态 -->
                        <div v-else-if="previewExpandedNodes.length === 0" class="py-4 text-center text-xs text-amber-600 dark:text-amber-400">
                            <span v-if="(localProfile.manualNodes || []).length === 0">
                                请先勾选手动节点，或点击「包含所有手动节点」查看展开结果
                            </span>
                            <span v-else>
                                当前选中的手动节点暂无优选展开（请检查优选配置是否启用）
                            </span>
                        </div>

                        <!-- 节点列表 -->
                        <div v-else class="max-h-48 space-y-1.5 overflow-y-auto pr-1">
                            <div
                                v-for="(node, idx) in previewExpandedNodes"
                                :key="idx"
                                class="flex items-center justify-between gap-2 rounded-lg bg-white px-3 py-2 dark:bg-gray-800"
                            >
                                <div class="min-w-0 flex-1">
                                    <p class="truncate font-mono text-xs font-medium text-gray-700 dark:text-gray-200">
                                        {{ node.expandedServer
                                        }}<span v-if="node.originalPort" class="text-gray-400">:{{ node.originalPort }}</span>
                                    </p>
                                    <p class="truncate text-xs text-gray-400 dark:text-gray-500">
                                        {{ node.originalName }}
                                        <span v-if="node.isGlobal" class="ml-1 rounded bg-green-100 px-1 text-green-600 dark:bg-green-900/40 dark:text-green-400">全局</span>
                                        · {{ node.configName }}
                                    </p>
                                </div>
                                <span class="shrink-0 text-xs text-gray-300 dark:text-gray-600">#{{ idx + 1 }}</span>
                            </div>
                        </div>

                        <p class="mt-3 text-xs text-amber-600 dark:text-amber-400">
                            💡 以上节点将在订阅被访问时由后端自动展开加入，无需额外配置
                        </p>
                    </div>
                </div>
            </div>
        </template>
    </Modal>
</template>
