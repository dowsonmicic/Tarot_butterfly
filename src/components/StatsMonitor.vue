<template>
  <div ref="container" class="stats-wrapper" :style="customStyle"></div>
</template>

<script setup>
/**
 * StatsMonitor.vue
 * 通用的性能监视组件，基于 stats.js
 */
import { onMounted, onUnmounted, ref, defineExpose, computed } from 'vue';
import Stats from 'three/examples/jsm/libs/stats.module.js';

const props = defineProps({
  // 显示模式：0: fps (每秒帧数), 1: ms (每帧耗时), 2: mb (内存占用)
  mode: {
    type: Number,
    default: 0
  },
  // 位置配置：top, bottom, left, right
  position: {
    type: Object,
    default: () => ({
      top: '10px',
      right: '10px'
    })
  }
});

const container = ref(null);
let stats = null;

// 计算自定义样式，用于覆盖 stats.js 默认的左上角定位
const customStyle = computed(() => {
  return {
    position: 'absolute',
    zIndex: 9999,
    ...props.position
  };
});

onMounted(() => {
  // 1. 初始化 Stats 实例
  stats = new Stats();
  
  // 2. 设置初始面板
  stats.showPanel(props.mode);
  
  // 3. 强制覆盖 stats.dom 的内联样式，以支持自定义定位
  // stats.js 默认会设置 left: 0px !important
  stats.dom.style.position = 'relative'; // 让它相对于我们的 wrapper 定位
  stats.dom.style.left = 'auto';
  stats.dom.style.top = 'auto';
  
  // 4. 将 stats 画布添加到容器中
  if (container.value) {
    container.value.appendChild(stats.dom);
  }
});

onUnmounted(() => {
  // 清理资源
  if (stats && stats.dom && stats.dom.parentNode) {
    stats.dom.parentNode.removeChild(stats.dom);
  }
  stats = null;
});

/**
 * 暴露 update 方法给父组件调用
 * 父组件需要在其 requestAnimationFrame 循环中调用此方法
 */
const update = () => {
  if (stats) {
    stats.update();
  }
};

/**
 * 暴露切换面板的方法
 */
const setMode = (mode) => {
  if (stats) {
    stats.showPanel(mode);
  }
};

defineExpose({
  update,
  setMode
});
</script>

<style scoped>
.stats-wrapper {
  /* 容器样式，实际位置由 props.position 控制 */
  pointer-events: none; /* 避免遮挡鼠标交互 */
}

/* 如果需要，可以在这里深度覆盖 stats.js 的 canvas 样式 */
:deep(canvas) {
  display: block !important;
}
</style>
