<!--
  ==================== 优选配置编辑模态框 ====================

  功能说明：
  - 编辑优选配置信息
  - 支持两种方式提供优选数据：
    1. 手动输入：直接填写域名/IP，每行一条
    2. 来源 URL：填写远程文件地址，后端订阅时实时拉取（推荐）
  - 两种方式可同时使用
  - 表单验证和提交

  ======================================================
-->

<script setup lang="ts">
import { computed, ref, watch } from 'vue';

import BaseModal from '../../../components/ui/BaseModal.vue';
import { useToastStore } from '../../../stores/toast';
import type { OptimalConfig } from '../../../types/index';

// ==================== Props & Emits ====================

const props = defineProps<{
    show: boolean;
    config: OptimalConfig | null;
    isNew: boolean;
}>();

const emit = defineEmits<{
    (e: 'update:show', value: boolean): void;
    (e: 'confirm', config: OptimalConfig): void;
}>();

// ==================== Store ====================

const { showToast } = useToastStore();

// ==================== Form State ====================

const formData = ref<Partial<OptimalConfig>>({
    name: '',
    description: '',
    items: [],
    type: 'domain',
    subscriptionIds: []
});

const itemsText = ref('');
const sourceUrlsText = ref('');
const formErrors = ref<Record<string, string>>({});

// ==================== Computed ====================

const isFormValid = computed(() => {
    const errors: Record<string, string> = {};

    if (!formData.value.name || formData.value.name.trim() === '') {
        errors.name = '配置名称不能为空';
    }

    const hasItems = itemsText.value
        .split('\n')
        .some((line) => line.trim() && !line.trim().startsWith('#'));

    const hasSourceUrls = sourceUrlsText.value.trim().length > 0;

    if (!hasItems && !hasSourceUrls) {
        errors.items = '请填写优选项列表，或填写来源 URL（后端订阅时自动拉取）';
    }

    formErrors.value = errors;
    return Object.keys(errors).length === 0;
});

const confirmDisabled = computed(() => !isFormValid.value);

// ==================== Methods ====================

const parseItems = () => {
    formData.value.items = itemsText.value
        .split('\n')
        .map((line) => line.trim())
        .filter((line) => line && !line.startsWith('#'));
};

const initializeForm = () => {
    if (props.config) {
        formData.value = JSON.parse(JSON.stringify(props.config));
        itemsText.value = (props.config.items || []).join('\n');
        sourceUrlsText.value = (props.config.sourceUrls || []).join('\n');
    } else {
        formData.value = {
            id: `opt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
            name: '',
            description: '',
            items: [],
            type: 'domain',
            subscriptionIds: [],
            createdAt: Date.now(),
            updatedAt: Date.now()
        };
        itemsText.value = '';
        sourceUrlsText.value = '';
    }
    formErrors.value = {};
};

const handleConfirm = () => {
    parseItems();

    if (!isFormValid.value) {
        const firstError = Object.values(formErrors.value)[0];
        if (firstError) showToast(`❌ ${firstError}`, 'error');
        return;
    }

    const sourceUrls = sourceUrlsText.value
        .split('\n')
        .map((u) => u.trim())
        .filter((u) => u.startsWith('http'));

    const config: OptimalConfig = {
        id: formData.value.id || `opt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
        name: (formData.value.name || '').trim(),
        description: formData.value.description
            ? (formData.value.description as string).trim()
            : undefined,
        items: formData.value.items || [],
        sourceUrls: sourceUrls.length > 0 ? sourceUrls : undefined,
        type: (formData.value.type as 'domain' | 'ip' | 'mixed') || 'domain',
        enabled: true,
        isGlobal: true,
        subscriptionIds: undefined,
        createdAt: (formData.value.createdAt as number) || Date.now(),
        updatedAt: Date.now()
    };

    emit('confirm', config);
};

const closeModal = () => {
    emit('update:show', false);
};

// ==================== Watchers ====================

watch(
    () => props.show,
    (newVal) => {
        if (newVal) initializeForm();
    }
);
</script>

<template>
    <BaseModal
        :show="props.show"
        size="2xl"
        :confirm-disabled="confirmDisabled"
        confirm-button-title="保存配置"
        @update:show="closeModal"
        @confirm="handleConfirm"
    >
        <template #title>
            <h3 class="text-lg font-bold text-gray-900 dark:text-white">
                {{ isNew ? '➕ 新增优选配置' : '✏️ 编辑优选配置' }}
            </h3>
        </template>

        <template #body>
            <div class="space-y-4">
                <!-- 配置名称 -->
                <div>
                    <label class="block text-sm font-semibold text-gray-700 dark:text-gray-300">
                        配置名称 <span class="text-red-500">*</span>
                    </label>
                    <input
                        v-model="formData.name"
                        type="text"
                        placeholder="例如: CDN域名优选"
                        class="mt-1 w-full rounded-lg border border-gray-300 px-3 py-2 text-sm transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white"
                    />
                    <p v-if="formErrors.name" class="mt-1 text-xs text-red-500">
                        {{ formErrors.name }}
                    </p>
                </div>

                <!-- 配置描述 -->
                <div>
                    <label class="block text-sm font-semibold text-gray-700 dark:text-gray-300">
                        配置描述
                    </label>
                    <textarea
                        v-model="formData.description"
                        placeholder="添加一些说明文字..."
                        rows="2"
                        class="mt-1 w-full rounded-lg border border-gray-300 px-3 py-2 text-sm transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white"
                    />
                </div>

                <!-- 配置类型 -->
                <div>
                    <label class="block text-sm font-semibold text-gray-700 dark:text-gray-300">
                        配置类型
                    </label>
                    <select
                        v-model="formData.type"
                        class="mt-1 w-full rounded-lg border border-gray-300 px-3 py-2 text-sm transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white"
                    >
                        <option value="domain">🌐 域名</option>
                        <option value="ip">📍 IP地址</option>
                        <option value="mixed">🔗 混合</option>
                    </select>
                </div>

                <!-- 来源 URL -->
                <div>
                    <label class="block text-sm font-semibold text-gray-700 dark:text-gray-300">
                        来源 URL
                        <span class="ml-1 text-xs font-normal text-gray-400">（推荐）</span>
                    </label>
                    <p class="mt-0.5 text-xs text-gray-500 dark:text-gray-400">
                        每行一个 URL。订阅被访问时后端实时拉取，远程更新无需任何操作。
                    </p>
                    <textarea
                        v-model="sourceUrlsText"
                        placeholder="https://raw.githubusercontent.com/example/repo/main/ips.txt"
                        rows="3"
                        class="mt-1 w-full rounded-lg border border-gray-300 px-3 py-2 font-mono text-xs transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white"
                    />
                </div>

                <!-- 手动优选项 -->
                <div>
                    <label class="block text-sm font-semibold text-gray-700 dark:text-gray-300">
                        手动优选项
                        <span class="ml-1 text-xs font-normal text-gray-400">（可选）</span>
                    </label>
                    <p class="mt-0.5 text-xs text-gray-500 dark:text-gray-400">
                        每行一条，支持 # 开头的注释行。与来源 URL 合并使用。
                    </p>
                    <textarea
                        v-model="itemsText"
                        :placeholder="formData.type === 'ip'
                            ? '192.168.1.1\n10.0.0.1\n# 这是注释'
                            : 'cdn.example.com\nproxy.example.com\n# 这是注释'"
                        rows="6"
                        class="mt-1 w-full rounded-lg border border-gray-300 px-3 py-2 font-mono text-xs transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white"
                    />
                    <p v-if="formErrors.items" class="mt-1 text-xs text-red-500">
                        {{ formErrors.items }}
                    </p>
                </div>

                <!-- 提示 -->
                <div
                    class="rounded-lg bg-blue-50 p-3 text-xs text-blue-700 dark:bg-blue-900/30 dark:text-blue-300"
                >
                    💡 填写来源 URL 后，每次订阅请求时后端会实时拉取最新数据，确保优选列表始终更新。
                </div>
            </div>
        </template>
    </BaseModal>
</template>
