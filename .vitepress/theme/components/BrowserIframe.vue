<template>
    <div class="browser-window" :class="{ minimized: isMinimized }">
        <!-- 标题栏 -->
        <div class="title-bar" @dblclick="toggleMaximize">
            <div v-if="!isWindowsLike" class="window-controls mac-style">
                <button class="control-btn close" @click.stop="doNothing" title="关闭">✕</button>
                <button class="control-btn minimize" @click.stop="toggleMinimize" title="最小化">—</button>
                <button class="control-btn maximize" @click.stop="openInNewTab" title="在新窗口打开">□</button>
            </div>
            <div v-if="isWindowsLike" class="spacer"></div>
            <div class="window-title" :class="{ 'windows-title': isWindowsLike }">
                {{ title }}
            </div>

            <div v-if="isWindowsLike" class="window-controls windows-style">
                <button class="control-btn minimize" @click.stop="toggleMinimize" title="最小化">—</button>
                <button class="control-btn maximize" @click.stop="openInNewTab" title="在新窗口打开">□</button>
                <button class="control-btn close" @click.stop="doNothing" title="关闭">✕</button>
            </div>

            <!-- 占位 flex 项，确保标题居中（macOS） -->
<!--            <div v-if="!isWindowsLike" class="spacer"></div>-->
        </div>

        <!-- 内容区（iframe） -->
        <div class="content-area" v-show="!isMinimized">
            <iframe
                :src="src"
                frameborder="0"
                class="browser-iframe"
                ref="iframeRef"
            ></iframe>
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const props = defineProps({
    src: {
        type: String,
        required: true
    },
    title: {
        type: String,
        default: 'page'
    }
})

const isMinimized = ref(false)
const iframeRef = ref(null)

// 检测是否为 Windows 或 Linux（按钮居右）
const isWindowsLike = ref(false)

onMounted(() => {
    const platform = navigator.userAgentData?.platform || navigator.platform
    // 更健壮的检测（兼容新旧 API）
    if (platform) {
        const lower = platform.toLowerCase()
        isWindowsLike.value = lower.includes('win') || lower.includes('linux')
    } else {
        // 回退：默认非 Windows（偏向 macOS 布局更安全）
        isWindowsLike.value = false
    }
})

const toggleMinimize = () => {
    isMinimized.value = !isMinimized.value
}

const openInNewTab = () => {
    window.open(props.src, '_blank')
}

const doNothing = () => {
    // 关闭按钮无功能
}

// 双击标题栏也触发最大化（打开新标签）
const toggleMaximize = () => {
    openInNewTab()
}
</script>

<style scoped>
.browser-window {
    width: 100%;
    height: 600px;
    border: 1px solid #ccc;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    display: flex;
    flex-direction: column;
    transition: height 0.3s ease;
}

.browser-window.minimized {
    height: 36px; /* 仅保留标题栏高度 */
}

.title-bar {
    height: 36px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 12px;
    user-select: none;
    cursor: default;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    font-size: 13px;
}

/* macOS 标题居中 */
.window-title {
    flex: 1;
    text-align: center;
}

/* Windows 标题靠左 */
.window-title.windows-title {
    text-align: left;
    margin-left: 12px;
}

.window-controls {
    display: flex;
    gap: 8px;
}

.spacer {
    flex: 1;
}

/* 实际控制按钮样式 */
.control-btn {
    width: 30px;
    height: 30px;
    border-radius: 50%;
    border: none;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: bold;
    cursor: pointer;
    outline: none;
    user-select: none;
}

.control-btn.minimize {
    background: #f5b630;
    color: #212529;
}
.control-btn.maximize {
    background: #2bc13e;
    color: white;
}
.control-btn.close {
    background: #f65c55;
    color: white;
}

.control-btn:hover {
    opacity: 0.9;
}

.content-area {
    flex: 1;
    width: 100%;
}

.browser-iframe {
    width: 100%;
    height: 100%;
}
</style>
