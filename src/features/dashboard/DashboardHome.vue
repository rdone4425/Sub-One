<!--
  仪表盘首页 - 显示系统概览和快捷操作
  - 励志语录轮播
  - 订阅源、节点、订阅组统计
  - 快捷添加按钮
  - Bento Grid 布局
-->

<script setup lang="ts">
import { computed, onMounted, ref } from 'vue';

import { storeToRefs } from 'pinia';

import NodeChart from '../../components/charts/NodeChart.vue';
import { useDataStore } from '../../stores/data';
import { useToastStore } from '../../stores/toast';

defineEmits<{
    (e: 'add-subscription'): void;
    (e: 'update-all-subscriptions'): void;
    (e: 'add-node'): void;
    (e: 'add-profile'): void;
}>();
const { showToast } = useToastStore();
const dataStore = useDataStore();
const { activeSubscriptions, manualNodes, profiles, totalNodeCount, activeNodeCount, optimalConfigs } =
    storeToRefs(dataStore);

// Computed for Display
const activeProfilesCount = computed(() => profiles.value.filter((p) => p.enabled).length);

// 优选配置统计
const enabledOptimalConfigsCount = computed(() =>
    optimalConfigs.value.filter((c) => c.enabled).length
);

const totalOptimalItems = computed(() => {
    const itemSet = new Set<string>();
    optimalConfigs.value
        .filter((c) => c.enabled)
        .forEach((c) => c.items.forEach((item) => { if (item.trim()) itemSet.add(item.trim()); }));
    return itemSet.size;
});

// 预计展开节点数：对每个启用的手动节点，按优选逻辑计算展开倍数
const expandedOptimalNodeCount = computed(() => {
    const enabledConfigs = optimalConfigs.value.filter(
        (c) => c.enabled !== false && Array.isArray(c.items) && c.items.length > 0
    );
    if (enabledConfigs.length === 0) {
        return manualNodes.value.filter((n) => n.enabled).length;
    }

    // 全局配置条目（去重）
    const globalItems = new Set<string>();
    enabledConfigs
        .filter((c) => c.isGlobal === true)
        .forEach((c) => c.items.forEach((item) => { if (item.trim()) globalItems.add(item.trim()); }));

    let total = 0;
    for (const node of manualNodes.value) {
        if (!node.enabled) continue;
        const specificIds = (node as any).optimalConfigIds as string[] | undefined;
        if (specificIds && specificIds.length > 0) {
            const specificItems = new Set<string>();
            for (const configId of specificIds) {
                const cfg = enabledConfigs.find((c) => c.id === configId);
                if (cfg?.items) cfg.items.forEach((item) => { if (item.trim()) specificItems.add(item.trim()); });
            }
            total += specificItems.size || 1;
        } else if (globalItems.size > 0) {
            total += globalItems.size;
        } else {
            total += 1;
        }
    }
    return total;
});
const isUpdatingAllSubs = ref(false);

const handleUpdateAll = async () => {
    isUpdatingAllSubs.value = true;
    const result = await dataStore.updateAllEnabledSubscriptions();
    isUpdatingAllSubs.value = false;

    if (result.success) {
        if (result.count && result.count > 0) {
            showToast(`✅ 成功更新 ${result.count} 个订阅`, 'success');
        } else {
            showToast('✅ 所有订阅已是最新状态', 'success');
        }
    } else {
        showToast('❌ ' + (result.message || '更新失败'), 'error');
    }
};

/** 励志语录数据库 */
const quotes = [
    {
        text: '成功不是终点，失败也不是终结，唯有勇气才是永恒。',
        author: '温斯顿·丘吉尔',
        category: '励志'
    },
    { text: '代码如诗，每一行都是对完美的追求。', author: '极客箴言', category: '技术' },
    { text: '今天的努力，是为了明天更好的自己。', author: '佚名', category: '励志' },
    {
        text: '优秀的程序员不是写代码最多的，而是删代码最多的。',
        author: '编程智慧',
        category: '技术'
    },
    { text: '保持简单，保持优雅，保持高效。', author: '设计哲学', category: '技术' },
    { text: '每一次调试，都是与bug的一场较量。', author: '程序员日常', category: '幽默' },
    { text: '不要害怕重构，害怕的应该是技术债。', author: '代码整洁之道', category: '技术' },
    {
        text: '真正的智慧不在于知道所有答案，而在于提出正确的问题。',
        author: '苏格拉底',
        category: '励志'
    },
    { text: '让代码自己说话，注释只是辅助。', author: 'Clean Code', category: '技术' },
    {
        text: 'bug不会因为你忽视它而消失，只会在生产环境中惊艳亮相。',
        author: '墨菲定律',
        category: '幽默'
    },
    { text: '持续学习，永不止步。今天比昨天更强大。', author: '成长心态', category: '励志' },
    { text: '好的架构不是设计出来的，而是演化出来的。', author: '架构之道', category: '技术' },
    { text: '测试不是负担，而是对代码的信心保障。', author: 'TDD实践', category: '技术' },
    { text: '编程不仅是科学，更是艺术。', author: 'Donald Knuth', category: '技术' },
    { text: '越简单的方案，越容易维护。', author: 'KISS原则', category: '技术' }
];

const currentQuote = ref(quotes[0]);
const isRefreshing = ref(false);

/** 获取随机语录（避免重复） */
const getRandomQuote = () => {
    let newQuote;
    do {
        newQuote = quotes[Math.floor(Math.random() * quotes.length)];
    } while (newQuote === currentQuote.value && quotes.length > 1);
    return newQuote;
};

/** 刷新语录 */
const refreshQuote = () => {
    isRefreshing.value = true;
    setTimeout(() => {
        currentQuote.value = getRandomQuote();
        isRefreshing.value = false;
    }, 300);
};

onMounted(() => {
    currentQuote.value = getRandomQuote();
});
</script>

<template>
    <div class="space-y-6">
        <!-- 励志语录卡片 -->
        <div
            class="card-glass group relative overflow-hidden rounded-3xl p-6 shadow-lg transition-all duration-500 hover:shadow-xl"
        >
            <div class="relative z-10">
                <!-- 标题栏 -->
                <div class="mb-4 flex items-center justify-between">
                    <div class="flex items-center gap-3">
                        <div
                            class="flex h-9 w-9 items-center justify-center rounded-xl border border-gray-300 bg-gray-100 backdrop-blur-md dark:border-gray-600 dark:bg-gray-700"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                class="h-5 w-5 text-purple-600 dark:text-purple-300"
                                fill="none"
                                viewBox="0 0 24 24"
                                stroke="currentColor"
                            >
                                <path
                                    stroke-linecap="round"
                                    stroke-linejoin="round"
                                    stroke-width="2"
                                    d="M9.663 17h4.673M12 3v1m6.364 1.636l-.707.707M21 12h-1M4 12H3m3.343-5.657l-.707-.707m2.828 9.9a5 5 0 117.072 0l-.548.547A3.374 3.374 0 0014 18.469V19a2 2 0 11-4 0v-.531c0-.895-.356-1.754-.988-2.386l-.548-.547z"
                                />
                            </svg>
                        </div>
                        <div>
                            <h3 class="text-base font-bold text-gray-800 dark:text-white">
                                每日一言
                            </h3>
                            <p class="text-xs text-gray-500 dark:text-gray-400">
                                Daily Inspiration
                            </p>
                        </div>
                    </div>

                    <!-- 分类标签和刷新按钮 -->
                    <div class="flex items-center gap-2">
                        <span
                            :class="{
                                'border-yellow-400/30 bg-yellow-400/20 text-yellow-700 dark:text-yellow-300':
                                    currentQuote.category === '励志',
                                'border-blue-400/30 bg-blue-400/20 text-blue-700 dark:text-blue-300':
                                    currentQuote.category === '技术',
                                'border-green-400/30 bg-green-400/20 text-green-700 dark:text-green-300':
                                    currentQuote.category === '幽默'
                            }"
                            class="rounded-lg border px-2.5 py-1 text-xs font-medium backdrop-blur-md transition-all duration-300"
                        >
                            {{ currentQuote.category }}
                        </span>

                        <button
                            :disabled="isRefreshing"
                            class="flex h-8 w-8 items-center justify-center rounded-lg border border-gray-300 bg-white/30 text-gray-700 backdrop-blur-md transition-all duration-300 hover:scale-110 hover:bg-white/50 active:scale-95 disabled:opacity-50 dark:border-gray-600 dark:bg-white/10 dark:text-gray-300 dark:hover:bg-white/20"
                            title="换一条"
                            @click="refreshQuote"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                class="h-4 w-4 transition-transform duration-500"
                                :class="{ 'rotate-180': isRefreshing }"
                                fill="none"
                                viewBox="0 0 24 24"
                                stroke="currentColor"
                            >
                                <path
                                    stroke-linecap="round"
                                    stroke-linejoin="round"
                                    stroke-width="2"
                                    d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"
                                />
                            </svg>
                        </button>
                    </div>
                </div>

                <!-- 语录内容 -->
                <div :key="currentQuote.text" class="quote-content animate-fadeIn">
                    <blockquote class="my-3">
                        <p
                            class="mb-2 text-center text-lg font-semibold leading-relaxed text-gray-800 dark:text-white md:text-xl"
                        >
                            "{{ currentQuote.text }}"
                        </p>
                        <footer class="flex items-center gap-2">
                            <div class="h-px flex-1 bg-gray-300/30 dark:bg-white/10"></div>
                            <cite
                                class="text-xs font-medium not-italic text-gray-600 dark:text-gray-400"
                            >
                                {{ currentQuote.author }}
                            </cite>
                            <div class="h-px flex-1 bg-gray-300/30 dark:bg-white/10"></div>
                        </footer>
                    </blockquote>
                </div>
            </div>
        </div>

        <!-- Bento Grid 布局 -->
        <div class="grid grid-cols-1 gap-4 md:grid-cols-3 lg:gap-6">
            <!-- 核心统计图表 (占2列) -->
            <div
                class="card-glass group relative flex min-h-75 flex-col overflow-hidden rounded-3xl p-6 shadow-xl md:col-span-2"
            >
                <div class="relative z-10 mb-6 flex items-center justify-between">
                    <div>
                        <h3 class="text-lg font-bold text-gray-900 dark:text-white">节点概览</h3>
                        <p class="text-xs text-gray-500 dark:text-gray-400">
                            Node Distribution & Status
                        </p>
                    </div>
                    <div class="flex gap-4">
                        <div class="text-right">
                            <div
                                class="text-[10px] font-bold uppercase tracking-wider text-gray-400"
                            >
                                活跃率
                            </div>
                            <div class="text-lg font-black text-indigo-600 dark:text-indigo-400">
                                {{
                                    totalNodeCount > 0
                                        ? Math.round((activeNodeCount / totalNodeCount) * 100)
                                        : 0
                                }}%
                            </div>
                        </div>
                    </div>
                </div>

                <div class="relative z-10 flex flex-1 flex-col items-center gap-6 sm:flex-row">
                    <!-- 图表区域 -->
                    <div class="h-55 w-full sm:w-1/2">
                        <NodeChart
                            :subscribed-nodes="totalNodeCount - manualNodes.length"
                            :manual-nodes="manualNodes.length"
                            :active-nodes="activeNodeCount"
                            :total-nodes="totalNodeCount"
                        />
                    </div>

                    <!-- 详细指标 -->
                    <div class="grid w-full grid-cols-2 gap-4 sm:w-1/2">
                        <div
                            class="rounded-2xl border border-gray-300 bg-white/50 p-4 shadow-sm dark:border-gray-700 dark:bg-black/20"
                        >
                            <div class="mb-1 text-xs text-gray-500 dark:text-gray-400">
                                活跃订阅
                            </div>
                            <div class="text-xl font-bold text-gray-900 dark:text-white">
                                {{ activeSubscriptions.length }}
                            </div>
                        </div>
                        <div
                            class="rounded-2xl border border-gray-300 bg-white/50 p-4 shadow-sm dark:border-gray-700 dark:bg-black/20"
                        >
                            <div class="mb-1 text-xs text-gray-500 dark:text-gray-400">
                                活跃节点
                            </div>
                            <div class="text-xl font-bold text-emerald-500">
                                {{ activeNodeCount }}
                            </div>
                        </div>
                        <div
                            class="rounded-2xl border border-gray-300 bg-white/50 p-4 shadow-sm dark:border-gray-700 dark:bg-black/20"
                        >
                            <div class="mb-1 text-xs text-gray-500 dark:text-gray-400">订阅组</div>
                            <div class="text-xl font-bold text-purple-500">
                                {{ profiles.length }}
                            </div>
                        </div>
                        <div
                            class="rounded-2xl border border-gray-300 bg-white/50 p-4 shadow-sm dark:border-gray-700 dark:bg-black/20"
                        >
                            <div class="mb-1 text-xs text-gray-500 dark:text-gray-400">
                                手动节点
                            </div>
                            <div class="text-xl font-bold text-orange-500">
                                {{ manualNodes.length }}
                            </div>
                        </div>
                    </div>
                </div>
            </div>

            <!-- 智能更新卡片 -->
            <div class="flex flex-col gap-4 lg:gap-6">
                <button
                    :disabled="isUpdatingAllSubs"
                    class="card-glass group relative flex flex-1 flex-col justify-between overflow-hidden rounded-3xl p-6 text-left shadow-lg transition-all duration-300 hover:shadow-xl"
                    @click="handleUpdateAll"
                >
                    <div
                        class="absolute inset-0 bg-linear-to-br from-blue-50 to-transparent opacity-0 transition-opacity duration-500 group-hover:opacity-100 dark:from-blue-900/10 dark:to-transparent"
                    ></div>
                    <div class="relative z-10">
                        <div
                            class="mb-4 flex h-12 w-12 items-center justify-center rounded-2xl bg-blue-500/10 text-blue-600 transition-transform duration-300 group-hover:scale-110 dark:bg-blue-500/20 dark:text-blue-400"
                        >
                            <svg
                                v-if="!isUpdatingAllSubs"
                                xmlns="http://www.w3.org/2000/svg"
                                class="h-6 w-6"
                                fill="none"
                                viewBox="0 0 24 24"
                                stroke="currentColor"
                            >
                                <path
                                    stroke-linecap="round"
                                    stroke-linejoin="round"
                                    stroke-width="2"
                                    d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"
                                />
                            </svg>
                            <svg
                                v-else
                                class="h-6 w-6 animate-spin"
                                xmlns="http://www.w3.org/2000/svg"
                                fill="none"
                                viewBox="0 0 24 24"
                            >
                                <circle
                                    class="opacity-25"
                                    cx="12"
                                    cy="12"
                                    r="10"
                                    stroke="currentColor"
                                    stroke-width="4"
                                ></circle>
                                <path
                                    class="opacity-75"
                                    fill="currentColor"
                                    d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
                                ></path>
                            </svg>
                        </div>
                        <h3 class="mb-1 text-xl font-bold text-gray-900 dark:text-white">
                            {{ isUpdatingAllSubs ? '正在更新...' : '立即更新' }}
                        </h3>
                        <p class="text-sm text-gray-500 dark:text-gray-400">
                            {{
                                isUpdatingAllSubs
                                    ? '正在同步最新节点信息'
                                    : '同步所有订阅源的节点信息'
                            }}
                        </p>
                    </div>
                </button>

                <!-- 订阅组指示 -->
                <div class="card-glass relative flex-1 rounded-3xl p-6 shadow-md">
                    <div class="mb-2 flex items-start justify-between">
                        <div
                            class="flex h-10 w-10 items-center justify-center rounded-xl bg-purple-500/20 text-purple-600 dark:text-purple-400"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                class="h-6 w-6"
                                fill="none"
                                viewBox="0 0 24 24"
                                stroke="currentColor"
                            >
                                <path
                                    stroke-linecap="round"
                                    stroke-linejoin="round"
                                    stroke-width="2"
                                    d="M17 20h5v-2a3 3 0 00-5.356-1.857M17 20H7m10 0v-2c0-.656-.126-1.283-.356-1.857M7 20H2v-2a3 3 0 015.356-1.857M7 20v-2c0-.656.126-1.283.356-1.857m0 0a5.002 5.002 0 019.288 0M15 7a3 3 0 11-6 0 3 3 0 016 0zm6 3a2 2 0 11-4 0 2 2 0 014 0zM7 10a2 2 0 11-4 0 2 2 0 014 0z"
                                />
                            </svg>
                        </div>
                        <span
                            class="rounded-full bg-purple-500 px-2 py-0.5 text-[10px] font-bold text-white"
                            >{{ activeProfilesCount }} Active</span
                        >
                    </div>
                    <div class="text-2xl font-black text-gray-900 dark:text-white">
                        {{ profiles.length }}
                        <span class="text-sm font-normal text-gray-500">订阅组</span>
                    </div>
                    <!-- 迷你进度条 -->
                    <div
                        class="mt-4 h-1.5 w-full overflow-hidden rounded-full bg-gray-200 dark:bg-white/10"
                    >
                        <div
                            class="h-full rounded-full bg-purple-500 transition-all duration-1000"
                            :style="{
                                width:
                                    profiles.length > 0
                                        ? `${(activeProfilesCount / profiles.length) * 100}%`
                                        : '0%'
                            }"
                        ></div>
                    </div>
                </div>
            </div>

            <!-- 快捷操作按钮 (3个) -->
            <button
                class="card-glass group flex min-h-30 items-center gap-4 rounded-2xl p-6 shadow-sm transition-all duration-300 hover:border-indigo-200 hover:shadow-md dark:hover:border-indigo-800"
                @click="$emit('add-subscription')"
            >
                <div
                    class="flex h-12 w-12 items-center justify-center rounded-full bg-gray-50 text-gray-400 transition-colors duration-300 group-hover:bg-indigo-500 group-hover:text-white dark:bg-gray-700"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-6 w-6"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                    >
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M12 4v16m8-8H4"
                        />
                    </svg>
                </div>
                <div class="text-left">
                    <p
                        class="font-bold text-gray-900 transition-colors group-hover:text-indigo-600 dark:text-white dark:group-hover:text-indigo-400"
                    >
                        添加订阅
                    </p>
                    <p class="text-xs text-gray-500 dark:text-gray-400">支持 HTTP/HTTPS</p>
                </div>
            </button>

            <button
                class="card-glass group flex min-h-30 items-center gap-4 rounded-2xl p-6 shadow-sm transition-all duration-300 hover:border-emerald-200 hover:shadow-md dark:hover:border-emerald-800"
                @click="$emit('add-node')"
            >
                <div
                    class="flex h-12 w-12 items-center justify-center rounded-full bg-gray-50 text-gray-400 transition-colors duration-300 group-hover:bg-emerald-500 group-hover:text-white dark:bg-gray-700"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-6 w-6"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                    >
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M13 10V3L4 14h7v7l9-11h-7z"
                        />
                    </svg>
                </div>
                <div class="text-left">
                    <p
                        class="font-bold text-gray-900 transition-colors group-hover:text-emerald-600 dark:text-white dark:group-hover:text-emerald-400"
                    >
                        添加节点
                    </p>
                    <p class="text-xs text-gray-500 dark:text-gray-400">支持多种协议</p>
                </div>
            </button>

            <button
                class="card-glass group flex min-h-30 items-center gap-4 rounded-2xl p-6 shadow-sm transition-all duration-300 hover:border-purple-200 hover:shadow-md dark:hover:border-purple-800"
                @click="$emit('add-profile')"
            >
                <div
                    class="flex h-12 w-12 items-center justify-center rounded-full bg-gray-50 text-gray-400 transition-colors duration-300 group-hover:bg-purple-500 group-hover:text-white dark:bg-gray-700"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-6 w-6"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                    >
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M3 7v10a2 2 0 002 2h14a2 2 0 002-2V9a2 2 0 00-2-2h-6l-2-2H5a2 2 0 00-2 2z"
                        />
                    </svg>
                </div>
                <div class="text-left">
                    <p
                        class="font-bold text-gray-900 transition-colors group-hover:text-purple-600 dark:text-white dark:group-hover:text-purple-400"
                    >
                        创建订阅组
                    </p>
                    <p class="text-xs text-gray-500 dark:text-gray-400">组合订阅和节点</p>
                </div>
            </button>

            <!-- 优选配置统计卡 -->
            <div
                class="card-glass md:col-span-3 relative overflow-hidden rounded-3xl p-5 shadow-md"
            >
                <div
                    class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between"
                >
                    <div class="flex items-center gap-3">
                        <div
                            class="flex h-10 w-10 flex-shrink-0 items-center justify-center rounded-xl bg-amber-500/20"
                        >
                            <span class="text-xl">🎯</span>
                        </div>
                        <div>
                            <h3 class="text-sm font-bold text-gray-900 dark:text-white">
                                优选配置
                            </h3>
                            <p class="text-xs text-gray-500 dark:text-gray-400">
                                Optimal Nodes Summary
                            </p>
                        </div>
                    </div>

                    <div class="flex items-center gap-6">
                        <div class="text-center">
                            <div class="text-2xl font-black text-amber-500">
                                {{ enabledOptimalConfigsCount }}
                            </div>
                            <div class="text-xs text-gray-500 dark:text-gray-400">启用配置</div>
                        </div>
                        <div
                            class="h-10 w-px self-center bg-gray-200 dark:bg-gray-700"
                        ></div>
                        <div class="text-center">
                            <div class="text-2xl font-black text-orange-500">
                                {{ totalOptimalItems }}
                            </div>
                            <div class="text-xs text-gray-500 dark:text-gray-400">优选条目</div>
                        </div>
                        <div
                            class="h-10 w-px self-center bg-gray-200 dark:bg-gray-700"
                        ></div>
                        <div class="text-center">
                            <div class="text-2xl font-black text-teal-500">
                                {{ expandedOptimalNodeCount }}
                            </div>
                            <div class="text-xs text-gray-500 dark:text-gray-400">
                                预计节点
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
/* 语录淡入动画 */
@keyframes fadeIn {
    from {
        opacity: 0;
        transform: translateY(10px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.animate-fadeIn {
    animation: fadeIn 0.5s ease-out;
}

.quote-content:hover {
    transform: scale(1.01);
    transition: transform 0.3s ease;
}
</style>
