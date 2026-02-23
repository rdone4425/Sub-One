<!--
  ==================== 优选配置卡片组件 ====================

  功能说明：
  - 显示单个优选配置信息
  - 显示配置名称、描述、项目数量
  - 支持启用/禁用切换
  - 支持编辑和删除操作
  - 支持展示项目预览

  =========================================================
-->

<script setup lang="ts">
import { computed } from 'vue';
import type { OptimalConfig } from '../../../types/index';

// ==================== Props & Emits ====================

const props = defineProps<{
    config: OptimalConfig;
}>();

const emit = defineEmits<{
    (e: 'edit'): void;
    (e: 'delete'): void;
    (e: 'toggle'): void;
}>();

// ==================== Computed ====================

const itemsPreview = computed(() => {
    const items = props.config.items || [];
    return items.slice(0, 3).join('\n');
});

const hasMoreItems = computed(() => {
    return (props.config.items || []).length > 3;
});

const itemsCount = computed(() => {
    return (props.config.items || []).length;
});

const statusIcon = computed(() => {
    if (!props.config.enabled) return '🔒';
    if (props.config.type === 'domain') return '🌐';
    if (props.config.type === 'ip') return '📍';
    return '🔗';
});

const typeLabel = computed(() => {
    const map = {
        domain: '域名',
        ip: 'IP地址',
        mixed: '混合'
    };
    return map[props.config.type] || props.config.type;
});
</script>

<template>
    <div
        class="group relative flex flex-col gap-3 rounded-2xl border border-gray-200 bg-white p-4 transition-all duration-300 hover:shadow-lg dark:border-gray-700 dark:bg-gray-800"
    >
        <!-- 启用状态指示 -->
        <div
            v-if="!config.enabled"
            class="absolute inset-0 rounded-2xl bg-gray-400/20 backdrop-blur-[1px]"
        />

        <!-- 头部：名称和状态 -->
        <div class="flex items-start justify-between gap-2">
            <div class="flex-1 pr-2">
                <h3 class="text-sm font-bold text-gray-900 dark:text-white">
                    {{ statusIcon }} {{ config.name }}
                </h3>
                <p
                    v-if="config.description"
                    class="mt-1 text-xs text-gray-500 dark:text-gray-400"
                >
                    {{ config.description }}
                </p>
            </div>

            <!-- 启用切换开关 -->
            <button
                :class="[
                    'relative h-6 w-10 rounded-full transition-all duration-300',
                    config.enabled
                        ? 'bg-blue-500 shadow-md'
                        : 'bg-gray-300 dark:bg-gray-600'
                ]"
                @click="$emit('toggle')"
            >
                <span
                    :class="[
                        'absolute top-0.5 h-5 w-5 rounded-full bg-white transition-all duration-300',
                        config.enabled ? 'right-0.5' : 'left-0.5'
                    ]"
                />
            </button>
        </div>

        <!-- 配置信息 -->
        <div class="flex gap-2 text-xs">
            <span
                class="inline-flex items-center gap-1 rounded-lg bg-blue-100 px-2 py-1 text-blue-700 dark:bg-blue-900 dark:text-blue-200"
            >
                {{ typeLabel }}
            </span>
            <span
                class="inline-flex items-center gap-1 rounded-lg bg-purple-100 px-2 py-1 text-purple-700 dark:bg-purple-900 dark:text-purple-200"
            >
                📊 {{ itemsCount }} 项
            </span>
        </div>

        <!-- 项目预览 -->
        <div
            v-if="itemsCount > 0"
            class="whitespace-pre-wrap rounded-lg bg-gray-50 p-2 text-xs font-mono text-gray-600 dark:bg-gray-700 dark:text-gray-300"
        >
            {{ itemsPreview }}
            <span v-if="hasMoreItems" class="block text-gray-500 dark:text-gray-400">
                ... 等 {{ itemsCount - 3 }} 项
            </span>
        </div>

        <!-- 底部操作按钮 -->
        <div class="flex gap-2 pt-2">
            <button
                class="flex-1 rounded-lg bg-gray-100 px-3 py-1.5 text-xs font-semibold text-gray-700 transition-all duration-300 hover:bg-gray-200 dark:bg-gray-700 dark:text-gray-200 dark:hover:bg-gray-600"
                @click="$emit('edit')"
            >
                ✏️ 编辑
            </button>
            <button
                class="flex-1 rounded-lg bg-red-100 px-3 py-1.5 text-xs font-semibold text-red-700 transition-all duration-300 hover:bg-red-200 dark:bg-red-900 dark:text-red-200 dark:hover:bg-red-800"
                @click="$emit('delete')"
            >
                🗑️ 删除
            </button>
        </div>

        <!-- 全局标签 -->
        <div v-if="config.isGlobal" class="pt-1">
            <span
                class="inline-flex items-center gap-1 rounded-lg bg-green-100 px-2 py-0.5 text-xs font-semibold text-green-700 dark:bg-green-900 dark:text-green-200"
            >
                🌍 全局配置
            </span>
        </div>
    </div>
</template>
