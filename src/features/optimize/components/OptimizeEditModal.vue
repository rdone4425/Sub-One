<!--
  ==================== 优选配置编辑模态框 ====================

  功能说明：
  - 编辑优选配置信息
  - 编辑优选项列表（支持多行输入）
  - 选择配置类型（域名/IP/混合）
  - 设置全局或特定订阅配置
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

// ==================== Store & Utils ====================

const { showToast } = useToastStore();

// ==================== Form State ====================

const formData = ref<Partial<OptimalConfig>>({
    name: '',
    description: '',
    items: [],
    type: 'domain',
    enabled: true,
    isGlobal: true,
    subscriptionIds: []
});

const itemsText = ref('');
const formErrors = ref<Record<string, string>>({});
const urlInput = ref('');
const isFetchingUrl = ref(false);
const inputMode = ref<'manual' | 'url'>('manual');

// ==================== Computed ====================

const isFormValid = computed(() => {
    const errors: Record<string, string> = {};

    if (!formData.value.name || formData.value.name.trim() === '') {
        errors.name = '配置名称不能为空';
    }

    // 检查优选项：优先检查 itemsText，再检查 formData.value.items
    const hasItems =
        (itemsText.value &&
            itemsText.value
                .split('\n')
                .some(
                    (line) =>
                        line.trim() && !line.trim().startsWith('#')
                )) ||
        (formData.value.items && formData.value.items.length > 0);

    if (!hasItems) {
        errors.items = '至少需要添加一条优选项';
    }

    formErrors.value = errors;
    return Object.keys(errors).length === 0;
});

const confirmDisabled = computed(() => !isFormValid.value);

// ==================== Methods ====================

const parseItems = () => {
    const items = itemsText.value
        .split('\n')
        .map((line) => line.trim())
        .filter((line) => line && !line.startsWith('#'));

    formData.value.items = items;

    // 更新表单验证状态
    formErrors.value = {};
};

/**
 * 从 URL 获取优选项列表
 */
const fetchFromUrl = async () => {
    if (!urlInput.value.trim()) {
        showToast('❌ 请输入 URL 地址', 'error');
        return;
    }

    isFetchingUrl.value = true;
    try {
        const urls = urlInput.value
            .split(',')
            .map((u) => u.trim())
            .filter((u) => u);

        const allItems: string[] = [];
        const itemSet = new Set<string>(); // 用于去重

        for (let i = 0; i < urls.length; i++) {
            const url = urls[i];
            try {
                console.log(`📥 [${i + 1}/${urls.length}] 正在获取: ${url}`);
                const response = await fetch(url);

                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}`);
                }

                const content = await response.text();
                const lines = content.split('\n');

                let addedCount = 0;
                lines.forEach((line) => {
                    const trimmed = line.trim();
                    // 过滤空行和注释
                    if (trimmed && !trimmed.startsWith('#')) {
                        // 移除注释部分
                        const item = trimmed.split('#')[0].trim();
                        if (item && !itemSet.has(item)) {
                            allItems.push(item);
                            itemSet.add(item);
                            addedCount++;
                        }
                    }
                });

                console.log(`✅ [${i + 1}/${urls.length}] 成功获取 ${addedCount} 个项目`);
            } catch (error) {
                const errorMsg = error instanceof Error ? error.message : '未知错误';
                console.error(`❌ [${i + 1}/${urls.length}] 获取失败: ${errorMsg}`);
                showToast(`❌ URL ${i + 1} 获取失败: ${errorMsg}`, 'error');
            }
        }

        if (allItems.length > 0) {
            // 追加到现有内容
            const existingLines = itemsText.value
                .split('\n')
                .map((line) => line.trim())
                .filter((line) => line && !line.startsWith('#'));

            const combinedItems = Array.from(
                new Set([...existingLines, ...allItems])
            );

            itemsText.value = combinedItems.join('\n');
            showToast(
                `✅ 成功获取 ${allItems.length} 个项目 (已去重)`,
                'success'
            );
            urlInput.value = '';
            inputMode.value = 'manual';
        } else {
            showToast('❌ 未获取到有效项目，请检查 URL', 'error');
        }
    } catch (error) {
        const errorMsg = error instanceof Error ? error.message : '未知错误';
        showToast(`❌ 获取失败: ${errorMsg}`, 'error');
    } finally {
        isFetchingUrl.value = false;
    }
};

const resetUrlInput = () => {
    urlInput.value = '';
    inputMode.value = 'manual';
};

const initializeForm = () => {
    if (props.config) {
        formData.value = JSON.parse(JSON.stringify(props.config));
        itemsText.value = (props.config.items || []).join('\n');
    } else {
        formData.value = {
            id: `opt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
            name: '',
            description: '',
            items: [],
            type: 'domain',
            enabled: true,
            isGlobal: true,
            subscriptionIds: [],
            createdAt: Date.now(),
            updatedAt: Date.now()
        };
        itemsText.value = '';
    }
    formErrors.value = {};
    urlInput.value = '';
    inputMode.value = 'manual';
};

const handleConfirm = () => {
    parseItems();

    if (!isFormValid.value) {
        const firstError = Object.values(formErrors.value)[0];
        if (firstError) {
            showToast(`❌ ${firstError}`, 'error');
        }
        return;
    }

    const config: OptimalConfig = {
        id: formData.value.id || `opt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
        name: (formData.value.name || '').trim(),
        description: formData.value.description ? (formData.value.description as string).trim() : undefined,
        items: formData.value.items || [],
        sourceUrls:
            inputMode.value === 'url' && urlInput.value.trim()
                ? urlInput.value
                      .split(',')
                      .map((u) => u.trim())
                      .filter((u) => u)
                : (formData.value.sourceUrls && formData.value.sourceUrls.length > 0
                    ? formData.value.sourceUrls
                    : undefined),
        type: (formData.value.type as 'domain' | 'ip' | 'mixed') || 'domain',
        enabled: formData.value.enabled !== undefined ? formData.value.enabled : true,
        isGlobal: formData.value.isGlobal !== undefined ? formData.value.isGlobal : true,
        subscriptionIds: formData.value.subscriptionIds && formData.value.subscriptionIds.length > 0
            ? formData.value.subscriptionIds
            : undefined,
        createdAt: (formData.value.createdAt as number) || Date.now(),
        updatedAt: Date.now()
    };

    emit('confirm', config);
};

const closeModal = () => {
    emit('update:show', false);
    resetUrlInput();
};

// ==================== Watchers ====================

watch(
    () => props.show,
    (newVal) => {
        if (newVal) {
            initializeForm();
        }
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
                <div class="grid grid-cols-2 gap-4">
                    <div>
                        <label class="block text-sm font-semibold text-gray-700 dark:text-gray-300">
                            配置类型 <span class="text-red-500">*</span>
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

                    <div>
                        <label class="block text-sm font-semibold text-gray-700 dark:text-gray-300">
                            配置范围
                        </label>
                        <select
                            v-model="formData.isGlobal"
                            class="mt-1 w-full rounded-lg border border-gray-300 px-3 py-2 text-sm transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white"
                        >
                            <option :value="true">🌍 全局配置</option>
                            <option :value="false">📌 特定订阅</option>
                        </select>
                    </div>
                </div>

                <!-- 优选项列表 -->
                <div>
                    <div class="flex items-center justify-between">
                        <label class="block text-sm font-semibold text-gray-700 dark:text-gray-300">
                            优选项列表 <span class="text-red-500">*</span>
                        </label>
                        <div class="flex gap-2">
                            <button
                                :class="[
                                    'text-xs font-semibold px-2 py-1 rounded-lg transition-all',
                                    inputMode.value === 'manual'
                                        ? 'bg-blue-500 text-white'
                                        : 'bg-gray-200 text-gray-700 dark:bg-gray-700 dark:text-gray-300'
                                ]"
                                @click="inputMode = 'manual'"
                            >
                                📝 手动输入
                            </button>
                            <button
                                :class="[
                                    'text-xs font-semibold px-2 py-1 rounded-lg transition-all',
                                    inputMode.value === 'url'
                                        ? 'bg-blue-500 text-white'
                                        : 'bg-gray-200 text-gray-700 dark:bg-gray-700 dark:text-gray-300'
                                ]"
                                @click="inputMode = 'url'"
                            >
                                🔗 远程获取
                            </button>
                        </div>
                    </div>

                    <!-- 手动输入模式 -->
                    <div v-if="inputMode === 'manual'" class="mt-2 space-y-2">
                        <p class="text-xs text-gray-500 dark:text-gray-400">
                            每行一条（支持注释，以 # 开头）
                        </p>
                        <textarea
                            v-model="itemsText"
                            :placeholder="`${formData.type === 'ip' ? '192.168.1.1\n10.0.0.1' : formData.type === 'domain' ? 'cdn.example.com\nproxy.example.com' : 'cdn.example.com\n192.168.1.1'}\n# 这是注释`"
                            rows="8"
                            class="w-full rounded-lg border border-gray-300 px-3 py-2 font-mono text-xs transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white"
                        />
                    </div>

                    <!-- 远程获取模式 -->
                    <div v-if="inputMode === 'url'" class="mt-2 space-y-2">
                        <p class="text-xs text-gray-500 dark:text-gray-400">
                            支持多个 URL，用逗号分隔。将自动提取有效项并去重。
                        </p>
                        <div class="flex gap-2">
                            <input
                                v-model="urlInput"
                                type="text"
                                placeholder="https://raw.githubusercontent.com/rdone4425/node/refs/heads/main/cf_ips.txt"
                                class="flex-1 rounded-lg border border-gray-300 px-3 py-2 text-sm transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white"
                                :disabled="isFetchingUrl"
                            />
                            <button
                                :disabled="isFetchingUrl || !urlInput.trim()"
                                :class="[
                                    'rounded-lg px-4 py-2 text-sm font-semibold transition-all',
                                    isFetchingUrl || !urlInput.trim()
                                        ? 'cursor-not-allowed bg-gray-300 text-gray-500 dark:bg-gray-700'
                                        : 'bg-blue-500 text-white hover:bg-blue-600'
                                ]"
                                @click="fetchFromUrl"
                            >
                                {{ isFetchingUrl ? '⏳ 获取中...' : '📥 获取' }}
                            </button>
                        </div>

                        <!-- 获取后的显示 -->
                        <div
                            v-if="itemsText"
                            class="mt-2 rounded-lg bg-green-50 p-3 dark:bg-green-900/20"
                        >
                            <p class="text-xs font-semibold text-green-700 dark:text-green-200">
                                ✅ 已获取内容，下方显示预览
                            </p>
                        </div>

                        <!-- 预览和编辑 -->
                        <textarea
                            v-model="itemsText"
                            placeholder="获取成功后在此显示"
                            rows="6"
                            class="w-full rounded-lg border border-gray-300 px-3 py-2 font-mono text-xs transition-colors focus:border-blue-500 focus:outline-none focus:ring-1 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-800 dark:text-white"
                        />
                    </div>

                    <!-- 错误提示 -->
                    <p v-if="formErrors.items" class="mt-1 text-xs text-red-500">
                        {{ formErrors.items }}
                    </p>

                    <!-- 项目统计 -->
                    <p class="mt-1 text-xs text-gray-500 dark:text-gray-400">
                        当前已添加:
                        <span class="font-semibold text-gray-700 dark:text-gray-300">
                            {{ formData.items?.length || 0 }}
                        </span>
                        项
                    </p>
                </div>

                <!-- 启用状态 -->
                <div class="flex items-center gap-2">
                    <input
                        :id="`enabled-${Date.now()}`"
                        v-model="formData.enabled"
                        type="checkbox"
                        class="h-4 w-4 cursor-pointer rounded border-gray-300"
                    />
                    <label
                        :for="`enabled-${Date.now()}`"
                        class="cursor-pointer text-sm font-medium text-gray-700 dark:text-gray-300"
                    >
                        ✅ 启用此配置
                    </label>
                </div>

                <!-- 提示信息 -->
                <div
                    class="rounded-lg bg-blue-50 p-3 text-xs text-blue-700 dark:bg-blue-900 dark:text-blue-200"
                >
                    💡 优选配置可用于在节点转换时替换服务器地址，提升访问速度和稳定性。
                </div>
            </div>
        </template>
    </BaseModal>
</template>
