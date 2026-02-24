<!--
  ==================== 优选节点卡片（只读展示）====================

  功能说明：
  - 展示由手动节点 + 优选配置 展开生成的虚拟节点
  - 外观与 ManualNodeCard 保持一致
  - 只读，无编辑/删除操作
  - 显示来源配置和来源节点信息

  ================================================================
-->

<script setup lang="ts">
import { computed } from 'vue';
import { useToastStore } from '../../../stores/toast';
import { getProtocolInfo } from '../../../utils/protocols';
import { copyToClipboard } from '../../../utils/utils';

const props = defineProps<{
    expandedServer: string;
    originalPort: string | number;
    originalName: string;
    protocol: string;
    configName: string;
    isGlobal: boolean;
}>();

const toastStore = useToastStore();

const protocolInfo = computed(() => getProtocolInfo(props.protocol.toLowerCase()));

const handleCopy = async () => {
    const success = await copyToClipboard(props.expandedServer);
    if (success) {
        toastStore.showToast('📋 已复制服务器地址', 'success');
    } else {
        toastStore.showToast('❌ 复制失败', 'error');
    }
};
</script>

<template>
    <!-- 卡片容器 -->
    <div
        class="card-glass group relative flex h-full flex-col overflow-hidden rounded-2xl border border-gray-300 transition-all duration-300 hover:-translate-y-1 hover:shadow-xl hover:shadow-amber-500/10 dark:border-gray-800"
    >
        <!-- 顶部彩色条 -->
        <div class="h-1.5 w-full bg-linear-to-r opacity-80" :class="protocolInfo.gradient"></div>

        <div class="flex flex-1 flex-col p-5">
            <!-- 头部：协议标签 + 优选标签 -->
            <div class="mb-4 flex items-center justify-between gap-2">
                <div class="flex flex-wrap items-center gap-2 overflow-hidden min-w-0">
                    <!-- 协议标签 -->
                    <span
                        class="inline-flex items-center gap-1.5 rounded-full border px-2.5 py-1 text-xs font-bold uppercase tracking-wide shadow-sm shrink-0"
                        :class="[
                            protocolInfo.bg,
                            protocolInfo.color,
                            'border-transparent bg-opacity-10 dark:bg-opacity-20'
                        ]"
                    >
                        <span class="text-sm font-normal drop-shadow-sm filter">{{ protocolInfo.icon }}</span>
                        <span>{{ protocolInfo.text }}</span>
                    </span>

                    <!-- 优选来源标签 -->
                    <span
                        class="inline-flex items-center gap-1 rounded-full border border-transparent bg-amber-100 bg-opacity-10 px-2.5 py-1 text-xs font-bold uppercase tracking-wide text-amber-700 shadow-sm shrink-0 dark:bg-amber-900/30 dark:text-amber-300"
                        :title="`优选配置: ${configName}`"
                    >
                        <span class="text-sm font-normal">🎯</span>
                        <span>{{ isGlobal ? '全局' : '优选' }}</span>
                    </span>
                </div>

                <!-- 右侧：复制按钮 -->
                <div class="flex items-center gap-1 opacity-100 lg:translate-x-2 lg:opacity-0 lg:group-hover:translate-x-0 lg:group-hover:opacity-100 transition-all duration-200">
                    <button
                        class="rounded-lg p-1.5 text-gray-400 transition-colors hover:bg-amber-50 hover:text-amber-600 dark:hover:bg-amber-900/40 dark:hover:text-amber-400"
                        title="复制服务器地址"
                        @click.stop="handleCopy"
                    >
                        <svg class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                            <path stroke-linecap="round" stroke-linejoin="round" d="M8 7v8a2 2 0 002 2h6M8 7V5a2 2 0 012-2h4.586a1 1 0 01.707.293l4.414 4.414a1 1 0 01.293.707V15a2 2 0 01-2 2h-2M8 7H6a2 2 0 01-2-2V3" />
                        </svg>
                    </button>
                </div>
            </div>

            <!-- 节点名称（原始节点名）-->
            <div class="mb-4 flex items-center justify-between gap-2">
                <h4
                    class="line-clamp-2 wrap-break-word text-base font-bold leading-snug text-gray-800 transition-all duration-300 hover:line-clamp-none dark:text-gray-100"
                    :title="originalName"
                >
                    {{ originalName }}
                </h4>
                <!-- 配置名称标签 -->
                <div class="flex-shrink-0 inline-flex items-center gap-1 rounded-lg bg-amber-100 px-2 py-1 text-xs font-semibold text-amber-700 dark:bg-amber-900/30 dark:text-amber-300">
                    <span>🎯</span>
                    <span class="truncate max-w-20" :title="configName">{{ configName }}</span>
                </div>
            </div>

            <!-- 底部信息：展开后的服务器地址 -->
            <div class="mt-auto border-t border-gray-300 pt-3 dark:border-gray-700/50">
                <div class="flex items-center justify-between gap-2 text-xs">
                    <div
                        class="flex items-center gap-1.5 overflow-hidden text-gray-500 dark:text-gray-400"
                        title="优选服务器地址"
                    >
                        <svg class="h-3.5 w-3.5 shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 12h14M12 5l7 7-7 7" />
                        </svg>
                        <span class="truncate font-mono">{{ expandedServer }}</span>
                        <span v-if="originalPort" class="text-gray-300 dark:text-gray-600">:</span>
                        <span v-if="originalPort" class="font-mono text-gray-400">{{ originalPort }}</span>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
