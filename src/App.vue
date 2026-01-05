<template>
  <div id="app-container">
    <!-- 根据 currentView 切换显示的组件 -->
    <Text v-if="currentView === 'text'" @back="currentView = 'tarot'" />
    <TarotScene v-else-if="currentView === 'tarot'" />
    <TestCube v-else-if="currentView === 'cube'" />
    <Butterfly v-else-if="currentView === 'butterfly'" />

    <!-- 底部导航按钮 -->
    <div class="debug-nav">
      <button :class="{ active: currentView === 'tarot' }" @click="currentView = 'tarot'">塔罗场景</button>
      <button :class="{ active: currentView === 'text' }" @click="currentView = 'text'">文字演示</button>
      <button :class="{ active: currentView === 'cube' }" @click="currentView = 'cube'">测试立方体</button>
      <button :class="{ active: currentView === 'butterfly' }" @click="currentView = 'butterfly'">蓝蝴蝶 (GLTF)</button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import TarotScene from './components/TarotScene.vue'
import TestCube from './components/TestCube.vue'
import Text from './components/Text.vue'
import Butterfly from './components/Butterfly.vue'

// 使用字符串标记当前视图，更易于扩展
// 可选值: 'tarot', 'cube', 'text', 'butterfly'
const currentView = ref('butterfly') // 默认显示新加载的蝴蝶模型
</script>

<style>
body {
  margin: 0;
  padding: 0;
  overflow: hidden;
  background-color: #050510;
}

#app {
  width: 100vw;
  height: 100vh;
}

#app-container {
  width: 100%;
  height: 100%;
  position: relative;
}

.debug-nav {
  position: absolute;
  bottom: 30px;
  /* 调低一点，避免遮挡内容 */
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000;
  display: flex;
  gap: 10px;
  background: rgba(0, 0, 0, 0.5);
  padding: 10px;
  border-radius: 12px;
  backdrop-filter: blur(5px);
}

.debug-nav button {
  padding: 8px 16px;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(212, 175, 55, 0.5);
  color: #fff;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s;
}

.debug-nav button:hover {
  background: rgba(212, 175, 55, 0.3);
  border-color: #d4af37;
}

.debug-nav button.active {
  background: #d4af37;
  color: #000;
  border-color: #fff;
  box-shadow: 0 0 15px rgba(212, 175, 55, 0.5);
}
</style>
