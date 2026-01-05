<template>
  <!-- 
    主容器，动态绑定 immersive 类来实现“沉浸模式”的特殊样式（如全屏、隐藏非必要 UI）。
  -->
  <div :class="{ immersive: isImmersive }">
    <!-- 
      加载遮罩层 
      v-if="isLoading" 控制其显示与隐藏。
      当资源加载、Three.js 初始化时显示，完成后消失。
    -->
    <div v-if="isLoading" id="loader">
      <div class="spinner"></div>
      <p style="margin-top:20px; color:#aaa;">正在连接命运之线...</p>
    </div>

    <!-- 3D 场景的渲染容器 -->
    <div ref="canvasContainer" id="canvas-container"></div>

    <!-- 手势光标调试点（仅在调试模式或特定交互时显示） -->
    <div ref="cursorDebug" id="cursor-debug"></div>

    <!-- UI 交互层，位于 3D 场景之上 -->
    <div id="ui-layer">
      <!-- 顶部状态栏 -->
      <div class="top-bar">
        <div class="status-box">
          <h1>🔮 沉浸式塔罗</h1>
          <!-- 动态显示当前的操作状态或引导文字 -->
          <div class="status-text">{{ statusText }}</div>
          <!-- 切换 鼠标/手势 模式 -->
          <button @click="toggleMode" class="mode-switch">切换输入模式</button>
          <!-- 切换全屏沉浸模式 -->
          <button @click="toggleImmersive" class="mode-switch" style="margin-left:5px;">沉浸模式 (I)</button>
        </div>

        <!-- 历史记录面板：保存本次抽到的卡牌 -->
        <div id="history-panel">
          <div
            style="text-align:center; border-bottom:1px solid #555; margin-bottom:5px; padding-bottom:5px; color:#d4af37;">
            📜 命运记录
          </div>
          <div id="history-content">
            <div v-for="(item, index) in historyList" :key="index" class="history-item">
              <div class="history-name">{{ item.title }}</div>
              <div class="history-meaning">{{ item.meaning }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- 抽卡结果浮层：展示当前选中卡牌的详细含义 -->
      <div id="result-overlay" :style="{ opacity: result.show ? 1 : 0 }">
        <h2 class="result-title" :style="{ color: result.isReversed ? '#aaa' : '#d4af37' }">{{ result.title }}</h2>
        <div class="result-sub">{{ result.meaning }}</div>
      </div>

      <!-- 底部操作引导 -->
      <div class="bottom-hint">
        {{ guideText }}
      </div>
    </div>

    <!-- 卡牌名称揭示动画层 -->
    <div ref="cardNameReveal" id="card-name-reveal"
      :class="{ showing: result.showNameReveal, show: result.showSublimation }">
      {{ result.revealName }}
    </div>

    <!-- 摄像头预览区域（用于手势识别反馈） -->
    <div id="camera-preview">
      <!-- 绘制手部关键点的画布 -->
      <canvas ref="outputCanvas" id="output-canvas"></canvas>
      <!-- 显示识别出的手势名称 -->
      <div id="gesture-label">{{ gestureLabel }}</div>
    </div>

    <!-- 隐藏的视频输入源，供 MediaPipe 读取数据 -->
    <video ref="inputVideo" id="input-video"></video>
  </div>
</template>

<script setup>
/**
 * -------------------------------------------------------------------------
 * 🔮 沉浸式塔罗场景组件
 * -------------------------------------------------------------------------
 * 功能：
 * 1. 集成 Three.js 实现 3D 卡牌轮播与交互。
 * 2. 集成 MediaPipe Hands 实现无接触手势控制（滚动、抓取）。
 * 3. 支持鼠标模式作为备选交互方案。
 * 
 * 知识点：
 * - Composition API: 使用 ref 和 reactive 管理状态。
 * - Three.js: 负责 3D 渲染、动画、光影效果。
 * - MediaPipe: 负责实时手势识别，通过 WebGL 加速。
 */
import { ref, reactive, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';

// --- 数据配置 ---
// 22 张大阿卡纳牌的详细定义，包含正位 (u) 和逆位 (r) 的含义
const TAROT_DATA = [
  { name: "0 愚人", u: "新的开始，冒险，天真", r: "鲁莽，轻率，风险", imgId: 0 },
  { name: "I 魔术师", u: "创造力，技能，意志力", r: "欺骗，沟通不畅", imgId: 1 },
  { name: "II 女祭司", u: "直觉，潜意识，神秘", r: "隐藏动机，压抑直觉", imgId: 2 },
  { name: "III 皇后", u: "丰饶，母性，自然", r: "依赖，缺乏创意", imgId: 3 },
  { name: "IV 皇帝", u: "权威，结构，父性", r: "专制，僵化，软弱", imgId: 4 },
  { name: "V 教皇", u: "传统，信仰，社会准则", r: "束缚，叛逆，挑战权威", imgId: 5 },
  { name: "VI 恋人", u: "爱，和谐，选择", r: "失衡，不和，逃避责任", imgId: 6 },
  { name: "VII 战车", u: "胜利，意志，自律", r: "失控，方向不明，冲动", imgId: 7 },
  { name: "VIII 力量", u: "勇气，耐力，掌控感", r: "自我怀疑，软弱，滥用职权", imgId: 8 },
  { name: "IX 隐士", u: "冥想，孤独，寻求真理", r: "孤立，退缩，偏执", imgId: 9 },
  { name: "X 命运之轮", u: "改变，周期，好运", r: "坏运，抵抗变化，外部压力", imgId: 10 },
  { name: "XI 正义", u: "公平，诚实，因果", r: "不公，偏见，逃避惩罚", imgId: 11 },
  { name: "XII 倒吊人", u: "牺牲，换位思考，停滞", r: "无谓的牺牲，顽固，拖延", imgId: 12 },
  { name: "XIII 死神", u: "结束，转变，重生", r: "恐惧改变，停滞不前", imgId: 13 },
  { name: "XIV 节制", u: "平衡，节制，融合", r: "失衡，极端，缺乏视野", imgId: 14 },
  { name: "XV 恶魔", u: "束缚，欲望，成瘾", r: "解脱，觉醒，重新掌控", imgId: 15 },
  { name: "XVI 高塔", u: "剧变，觉醒，灾难", r: "逃避灾难，恐惧改变，内部动荡", imgId: 16 },
  { name: "XVII 星星", u: "希望，灵感，宁静", r: "失望，迷失，缺乏信心", imgId: 17 },
  { name: "XVIII 月亮", u: "幻想，不安，潜意识", r: "揭开谎言，克服恐惧，真相大白", imgId: 18 },
  { name: "XIX 太阳", u: "成功，快乐，活力", r: "消极，缺乏热情，虚假的快乐", imgId: 19 },
  { name: "XX 审判", u: "觉醒，重生，使命感", r: "自我怀疑，错失良机，逃避评判", imgId: 20 },
  { name: "XXI 世界", u: "完成，整合，旅行", r: "未完成，缺乏闭环", imgId: 21 }
];

// 资源服务器基础路径（GitHub 托管图片）
const IMG_BASE_URL = "https://raw.githubusercontent.com/dowsonmicic/Tarot_butterfly/refs/heads/images/docs/";
// 对应的文件名映射
const RWS_MAP = [
  "1.jpg", "2.jpg", "3.jpg", "4.jpg", "5.jpg", "6.jpg",
  "7.jpg", "8.jpg", "9.jpg", "10.jpg", "11.jpg", "12.jpg",
  "13.jpg", "14.jpg", "15.jpg", "16.jpg", "17.jpg", "18.jpg",
  "19.jpg", "20.jpg", "21.jpg", "22.jpg"
];
// 卡牌背面统一图片
const BACK_IMG_URL = IMG_BASE_URL + "bm.jpg";

// --- 响应式状态 (Vue Reactive State) ---
const isLoading = ref(true);         // 是否显示加载遮罩
const statusText = ref("初始化中..."); // UI 顶部的状态文本
const guideText = ref("初始化手势识别中..."); // UI 底部的引导提示
const historyList = ref([]);         // 历史抽卡记录
const isImmersive = ref(false);      // 是否开启沉浸模式
const gestureLabel = ref("未检测");    // 摄像头预览上的手势标签

// 抽卡结果的详细状态
const result = reactive({
  show: false,             // 是否显示结果文字
  title: "",               // 卡牌名称
  meaning: "",             // 卡牌含义
  isReversed: false,       // 是否是逆位
  revealName: "",          // 揭示动画中的名称
  showNameReveal: false,   // 是否显示大字揭示动画
  showSublimation: false   // 是否处于“升华”（消失）动画阶段
});

// --- DOM 引用 (Template Refs) ---
const canvasContainer = ref(null); // Three.js 画布容器
const cursorDebug = ref(null);     // 调试用的小红点
const outputCanvas = ref(null);    // MediaPipe 绘制关键点的画布
const inputVideo = ref(null);      // 摄像头视频源

// --- Three.js 核心变量 ---
let scene, camera, renderer, starfield, textureLoader, backTexture;
let starlights = [];               // 存放所有的粒子特效实例
let animationId;                   // requestAnimationFrame 的 ID
let cardGroup;                     // 专门存放卡牌的容器（Group），方便批量管理和清理
let currentLoadId = 0;             // 异步加载 ID，用于解决“异步竞态”问题（防止旧卡牌在清理后又冒出来）

/**
 * 游戏核心状态机
 * 管理交互模式、滚动速度、手势计数器等
 */
const gameState = reactive({
  mode: 'HAND',            // 当前模式：'HAND' (手势) 或 'MOUSE' (鼠标)
  cardMeshes: [],          // 当前场景中所有卡牌的网格对象
  cardDataList: [],        // 对应卡牌的业务数据
  scrollOffset: 0,         // 当前轮播图的偏移量
  scrollSpeed: 0.07,       // 基础滚动步长
  currentSpeed: 0.10,      // 动态计算后的滚动速度（根据手掌张开程度变化）
  phase: 'IDLE',           // 阶段：'IDLE' (闲置), 'LOADING' (加载), 'SCROLLING' (滚动), 'FLIPPING' (翻牌), 'SHOWING' (展示)
  selectedMesh: null,      // 当前被选中的卡牌网格
  selectedData: null,      // 当前被选中的卡牌数据
  selectedIndex: -1,       // 当前被选中的索引
  cursor: { x: 0, y: 0 },  // 交互坐标
  isHandOpen: false,       // 手掌是否张开（控制滚动）
  openness: 0,             // 手掌张开程度 (0-1)
  isFist: false,           // 是否握拳（触发抽取）
  wasFist: false,          // 上一帧是否握拳
  isDragging: false,       // 鼠标模式下是否正在拖拽
  fistCounter: 0,          // 握拳防抖计数器
  openCounter: 0,          // 张开防抖计数器
  gestureThreshold: 5,     // 防抖阈值（连续检测到 N 帧才认为手势生效）
  isHoldingFist: false,    // 是否正处于握拳选中状态
});

const CARD_COUNT = 22;       // 总卡牌数
const CARD_SPACING = 3.2;    // 卡牌之间的物理间距
const TOTAL_WIDTH = CARD_COUNT * CARD_SPACING; // 整个轮播图的总宽度

// --- 内部类：星空背景系统 ---
/**
 * 【详细解释星空系统逻辑】
 * 
 * 1. 结构：使用 THREE.Points (粒子系统) 渲染。
 * 2. 几何体：THREE.BufferGeometry，存储 2000 个点的位置、颜色和尺寸。
 * 3. 动画：在 update 方法中，每一帧都让星星在 Z 轴上向相机移动（模拟飞越星空感）。
 * 4. 循环：当星星飞过相机（Z > 20）时，将其重置到远方（Z = -30）。
 */
class Starfield {
  constructor() {
    this.count = 2000;       // 星星数量
    this.geometry = new THREE.BufferGeometry();
    const positions = new Float32Array(this.count * 3);
    const colors = new Float32Array(this.count * 3);
    const sizes = new Float32Array(this.count);

    for (let i = 0; i < this.count; i++) {
      // 随机分布在三维空间中
      positions[i * 3] = (Math.random() - 0.5) * 100;
      positions[i * 3 + 1] = (Math.random() - 0.5) * 100;
      positions[i * 3 + 2] = (Math.random() - 0.5) * 50 - 20;

      // 给星星随机的浅色调（蓝、白、黄）
      colors[i * 3] = 0.8 + Math.random() * 0.2;
      colors[i * 3 + 1] = 0.8 + Math.random() * 0.2;
      colors[i * 3 + 2] = 0.9 + Math.random() * 0.1;

      sizes[i] = Math.random() * 0.15 + 0.05;
    }

    this.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    this.geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    this.geometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

    this.material = new THREE.PointsMaterial({
      size: 0.1,
      vertexColors: true,
      transparent: true,
      opacity: 0.8,
      blending: THREE.AdditiveBlending, // 加法混合，让重叠的星星更亮
      sizeAttenuation: true             // 开启尺寸衰减，模拟远小近大
    });

    this.mesh = new THREE.Points(this.geometry, this.material);
    scene.add(this.mesh);
  }

  update() {
    const positions = this.geometry.attributes.position.array;
    const time = Date.now() * 0.0005;

    for (let i = 0; i < this.count; i++) {
      // 星星缓慢向相机飞来
      positions[i * 3 + 2] += 0.01;
      // 飞过相机后重置到远处，形成无限循环
      if (positions[i * 3 + 2] > 20) {
        positions[i * 3 + 2] = -30;
      }
    }
    this.geometry.attributes.position.needsUpdate = true;
    // 星星闪烁效果
    this.material.opacity = 0.6 + Math.sin(time) * 0.2;
  }
}

// --- 内部类：星光升华粒子系统 ---
/**
 * 【详细解释粒子特效逻辑】
 * 
 * 当用户抽中一张卡牌并选择“升华”时，卡牌会崩解为 2500 个上升的粒子。
 * 1. 初始位置：粒子随机分布在原卡牌的长方形区域内。
 * 2. 运动轨迹：每个粒子有自己的初速度，且在 update 中会加入 Sin 波动的扰动，模拟轻飘飘的灵感升腾感。
 * 3. 生命周期：每个粒子有 life 属性，随时间减少，同时 opacity 也会降低。
 * 4. 资源释放：当所有粒子都不可见时，会自动从 scene 中 remove 并调用 dispose 销毁几何体和材质，防止显存泄漏。
 */
class StarlightEffect {
  constructor(position) {
    this.particlesCount = 2500;
    this.geometry = new THREE.BufferGeometry();
    const positions = [];
    const velocities = [];
    const colors = [];
    const life = [];

    const w = 4.0, h = 5.5; // 卡牌的大致尺寸
    for (let i = 0; i < this.particlesCount; i++) {
      // 粒子初始位置限制在卡牌区域内
      positions.push(
        position.x + (Math.random() - 0.5) * w,
        position.y + (Math.random() - 0.5) * h,
        position.z + (Math.random() - 0.5) * 0.2
      );

      // 赋予粒子向上的速度
      const speed = 0.02 + Math.random() * 0.05;
      velocities.push(
        (Math.random() - 0.5) * 0.02,
        speed * 1.5 + 0.01,
        (Math.random() - 0.5) * 0.02
      );

      // 粒子颜色（金、白、紫）
      const colorChoice = Math.random();
      if (colorChoice < 0.5) {
        colors.push(1.0, 0.9, 0.5);
      } else if (colorChoice < 0.8) {
        colors.push(1.0, 1.0, 1.0);
      } else {
        colors.push(0.9, 0.7, 1.0);
      }

      life.push(1.0 + Math.random() * 1.2); // 粒子的寿命
    }

    this.geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    this.geometry.setAttribute('velocity', new THREE.Float32BufferAttribute(velocities, 3));
    this.geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));
    this.geometry.setAttribute('life', new THREE.Float32BufferAttribute(life, 1));

    this.material = new THREE.PointsMaterial({
      size: 0.12,
      transparent: true,
      opacity: 1,
      vertexColors: true,
      blending: THREE.AdditiveBlending,
      depthWrite: false
    });

    this.mesh = new THREE.Points(this.geometry, this.material);
    this.mesh.frustumCulled = false;
    this.startTime = Date.now();
    scene.add(this.mesh);
  }

  update() {
    const positions = this.geometry.attributes.position.array;
    const velocities = this.geometry.attributes.velocity.array;
    const lives = this.geometry.attributes.life.array;
    let alive = false;
    const elapsed = (Date.now() - this.startTime) * 0.001;

    for (let i = 0; i < this.particlesCount; i++) {
      if (lives[i] > 0) {
        alive = true;
        // 应用速度，让粒子飘起来
        positions[i * 3] += velocities[i * 3];
        positions[i * 3 + 1] += velocities[i * 3 + 1];
        positions[i * 3 + 2] += velocities[i * 3 + 2];

        // 加入正弦波动的效果，模拟随风飘动的灵感
        const wave = elapsed * 2 + i * 0.05;
        positions[i * 3] += Math.sin(wave) * 0.005;
        positions[i * 3 + 2] += Math.cos(wave) * 0.005;

        lives[i] -= 0.004; // 扣除寿命
      }
    }

    this.material.opacity = Math.max(0, this.material.opacity - 0.004);
    this.geometry.attributes.position.needsUpdate = true;

    // 粒子全部死掉后，从场景中移除自己，释放资源
    if (!alive || this.material.opacity <= 0) {
      scene.remove(this.mesh);
      this.geometry.dispose();
      this.material.dispose();
      return false;
    }
    return true;
  }
}

/**
 * 初始化 Three.js 核心环境
 */
const initThree = () => {
  // 1. 创建场景并设置指数雾（远处的卡牌会隐约消失在黑暗中）
  scene = new THREE.Scene();
  scene.fog = new THREE.FogExp2(0x050510, 0.03);

  // 2. 相机：视野 60 度
  camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.set(0, 0, 8);

  // 3. 渲染器：开启透明支持
  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  canvasContainer.value.appendChild(renderer.domElement);

  // 4. 灯光系统
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4); // 环境光：基础亮度
  scene.add(ambientLight);
  const dirLight = new THREE.DirectionalLight(0xffd700, 1.0);  // 平行光：模拟金色阳光
  dirLight.position.set(5, 10, 7);
  scene.add(dirLight);
  const pointLight = new THREE.PointLight(0xaa88ff, 0.8);     // 点光源：紫色魔法氛围
  pointLight.position.set(-5, 2, 3);
  scene.add(pointLight);

  // 5. 初始化星空和卡牌加载器
  starfield = new Starfield();
  textureLoader = new THREE.TextureLoader();
  textureLoader.crossOrigin = 'anonymous';
  backTexture = textureLoader.load(BACK_IMG_URL);

  // 6. 初始化卡牌容器（重要：解决卡牌残留的关键）
  // 我们不再直接把卡牌加进 scene，而是加进这个 cardGroup。
  // 每次重置场景时，只需要 cardGroup.clear() 就能确保旧卡牌彻底消失。
  cardGroup = new THREE.Group();
  scene.add(cardGroup);

  window.addEventListener('resize', onWindowResize);
};

const onWindowResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
};

/**
 * 核心逻辑：生成所有卡牌
 * 这是一个复杂的过程，包含洗牌、图片异步加载、错误处理等
 */
const spawnAllCards = () => {
  gameState.phase = 'LOADING';
  statusText.value = "状态: 正在召唤命运之牌...";

  // 增加加载序列 ID。
  // 知识点：这是为了处理“异步竞争”。如果在加载中用户又触发了一次 spawn，
  // 旧的加载回调（textureLoader.load）依然会执行，导致旧卡牌莫名其妙出现在新场景。
  // 通过比较 loadId，我们可以抛弃掉过期的加载结果。
  currentLoadId++;
  const thisLoadId = currentLoadId;

  // 【修复卡牌残留】彻底清理旧场景，释放显存
  if (cardGroup) {
    // 递归遍历 Group 中的所有子对象，手动释放资源
    cardGroup.traverse((child) => {
      if (child.isMesh) {
        child.geometry.dispose();
        if (Array.isArray(child.material)) {
          child.material.forEach(m => {
            if (m.map) m.map.dispose();
            m.dispose();
          });
        } else {
          if (child.material.map) child.material.map.dispose();
          child.material.dispose();
        }
      }
    });
    cardGroup.clear(); // 清空容器
  }

  // 重置状态
  gameState.cardMeshes = [];
  gameState.cardDataList = [];
  gameState.scrollOffset = 0;
  gameState.selectedMesh = null;
  gameState.selectedData = null;

  // 1. 生成洗牌后的索引序列 (Fisher-Yates Shuffle)
  const shuffledIndices = Array.from({ length: 22 }, (_, i) => i);
  for (let i = shuffledIndices.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [shuffledIndices[i], shuffledIndices[j]] = [shuffledIndices[j], shuffledIndices[i]];
  }

  // 2. 异步加载每一张卡牌的图片
  let loadedCount = 0;
  shuffledIndices.forEach((cardId, index) => {
    const cardData = TAROT_DATA[cardId];
    const isReversed = Math.random() < 0.5; // 50% 概率生成逆位
    const imgUrl = IMG_BASE_URL + RWS_MAP[cardData.imgId];

    gameState.cardDataList[index] = { ...cardData, isReversed };

    textureLoader.load(
      imgUrl,
      (tex) => {
        // 如果 ID 不匹配，说明这是被抛弃的旧请求，直接销毁纹理并返回
        if (thisLoadId !== currentLoadId) {
          tex.dispose();
          return;
        }
        createCarouselCard(tex, index, isReversed);
        loadedCount++;
        // 只有当所有卡牌都加载（或报错处理）完成后，才开始入场动画
        if (loadedCount === CARD_COUNT) onAllCardsLoaded(thisLoadId);
      },
      undefined,
      () => {
        // 加载失败时使用 Canvas 实时生成一个占位图，保证流程不中断
        if (thisLoadId !== currentLoadId) return;
        const placeholder = new THREE.CanvasTexture(createPlaceholderCanvas(cardData.name));
        createCarouselCard(placeholder, index, isReversed);
        loadedCount++;
        if (loadedCount === CARD_COUNT) onAllCardsLoaded(thisLoadId);
      }
    );
  });
};

/**
 * 创建单张卡牌的 3D 网格对象
 */
const createCarouselCard = (frontTex, index, isReversed) => {
  // 定义长方形薄片几何体
  const geometry = new THREE.BoxGeometry(2.5, 4.0, 0.05);
  frontTex.colorSpace = THREE.SRGBColorSpace;
  backTexture.colorSpace = THREE.SRGBColorSpace;

  // 设置不同面的材质：侧面灰色，正面卡面图，背面统一图
  const sideMat = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.5 });
  const frontMat = new THREE.MeshStandardMaterial({ map: frontTex, roughness: 0.4, metalness: 0.1 });
  const backMat = new THREE.MeshStandardMaterial({ map: backTexture, roughness: 0.6 });

  // Mesh 材质顺序：[x+, x-, y+, y-, z+, z-]。这里 z+ 是正面，z- 是背面
  const materials = [sideMat, sideMat, sideMat, sideMat, frontMat, backMat];
  const mesh = new THREE.Mesh(geometry, materials);

  // 将业务数据存入 userData 方便后续读取
  mesh.userData = { index, isReversed, originalIndex: index, frontTex };
  mesh.position.y = -10; // 初始位置在屏幕下方，等待入场动画升起
  mesh.position.z = -2;
  mesh.rotation.y = Math.PI; // 初始面向背面（神秘感）

  cardGroup.add(mesh); // 加入统一容器
  gameState.cardMeshes[index] = mesh;
};

/**
 * 入场动画：所有卡牌从下方平滑升起
 */
const onAllCardsLoaded = (loadId) => {
  if (loadId !== currentLoadId) return;

  let progress = 0;
  const intro = () => {
    // 动画过程中也要检查 loadId，如果用户中途刷新了，立即停止旧动画
    if (loadId !== currentLoadId) return;

    progress += 0.025;
    const easeOut = 1 - Math.pow(1 - Math.min(progress, 1), 3); // Cubic Ease Out 效果

    gameState.cardMeshes.forEach((mesh, i) => {
      if (!mesh) return;
      const targetX = (i - CARD_COUNT / 2) * CARD_SPACING;
      mesh.position.x = targetX;
      mesh.position.y = THREE.MathUtils.lerp(-10, 0, easeOut);
      mesh.position.z = -2;
      mesh.rotation.y = Math.PI;
      if (mesh.userData.isReversed) mesh.rotation.z = Math.PI; // 逆位旋转 180 度
    });

    if (progress < 1) {
      requestAnimationFrame(intro);
    } else {
      gameState.phase = 'IDLE';
      resetUIForNewRound();
      isLoading.value = false;
    }
  };
  intro();
};

/**
 * 创建卡牌加载失败时的文字占位图
 * 知识点：CanvasTexture 可以把 Canvas 变成 Three.js 的贴图
 */
const createPlaceholderCanvas = (text) => {
  const c = document.createElement('canvas');
  c.width = 256; c.height = 400;
  const ctx = c.getContext('2d');
  ctx.fillStyle = '#1a1a1a'; ctx.fillRect(0, 0, 256, 400);
  ctx.strokeStyle = '#d4af37'; ctx.lineWidth = 10; ctx.strokeRect(5, 5, 246, 390);
  ctx.fillStyle = '#d4af37'; ctx.font = 'bold 24px Arial'; ctx.textAlign = 'center';
  ctx.fillText(text, 128, 180);
  return c;
};

/**
 * 重置 UI 提示文字
 */
const resetUIForNewRound = () => {
  result.show = false;
  statusText.value = gameState.mode === 'HAND'
    ? "状态: 张开手掌开始滚动"
    : "状态: 按住左键开始滚动";
  guideText.value = gameState.mode === 'HAND'
    ? "🖐 张开=滚动 | ✊ 握拳=抽取 | 松开=升华"
    : "🖱 按住=滚动 | 双击=抽取 | 再次双击=升华";
};

/**
 * 轮播图位置更新逻辑（核心交互算法）
 * 
 * 1. 无限循环：通过计算 x 坐标并配合 while 循环实现超出边界后自动折返。
 * 2. 交互缩放：计算卡牌到屏幕中心（x=0）的距离，距离越近缩放越大、位置越靠前。
 * 3. 动态旋转：给卡牌一个随 x 坐标变化的 y 轴偏转，营造圆弧形的空间感。
 */
const updateCarousel = () => {
  gameState.cardMeshes.forEach((mesh, i) => {
    // 如果卡牌不存在，或者是当前被选中的那张（正在翻转），则不参与轮播逻辑
    if (!mesh || mesh === gameState.selectedMesh) return;

    // 计算基于 scrollOffset 的位置，并实现无限循环滚动
    let x = (i - CARD_COUNT / 2) * CARD_SPACING + gameState.scrollOffset;
    while (x > TOTAL_WIDTH / 2) x -= TOTAL_WIDTH;
    while (x < -TOTAL_WIDTH / 2) x += TOTAL_WIDTH;

    mesh.position.x = x;
    const distFromCenter = Math.abs(x);

    // 离中心越近，缩放越大，距离相机越近 (z 越大)
    const scale = THREE.MathUtils.lerp(1.15, 0.9, Math.min(distFromCenter / 8, 1));
    const zOffset = THREE.MathUtils.lerp(1, 0, Math.min(distFromCenter / 8, 1));

    // 使用 lerp 平滑插值，避免数值突变导致抖动
    mesh.scale.lerp(new THREE.Vector3(scale, scale, scale), 0.15);
    mesh.position.z = THREE.MathUtils.lerp(mesh.position.z, -2 + zOffset, 0.15);
    mesh.position.y = THREE.MathUtils.lerp(mesh.position.y, 0, 0.1);

    // 增加一点弧度旋转，模拟圆柱形排列感
    mesh.rotation.y = Math.PI + x * 0.05;
  });
};

/**
 * 获取当前最靠近屏幕中心的卡牌（即用户“指向”的牌）
 */
const getCenterCard = () => {
  let closest = null;
  let minDist = Infinity;
  gameState.cardMeshes.forEach((mesh, i) => {
    if (!mesh) return;
    const dist = Math.abs(mesh.position.x);
    if (dist < minDist) {
      minDist = dist;
      closest = { mesh, index: i, data: gameState.cardDataList[i] };
    }
  });
  return closest;
};

/**
 * -------------------------------------------------------------------------
 * 核心动画/更新循环 (Render Loop)
 * -------------------------------------------------------------------------
 * 知识点：这是 Three.js 的“心脏”，每秒执行约 60 次。
 */
const animate = () => {
  animationId = requestAnimationFrame(animate);

  // 1. 更新粒子特效和星空
  if (starfield) starfield.update();
  starlights = starlights.filter(s => s.update());

  // 2. 交互状态处理
  if (gameState.cardMeshes.length > 0 && gameState.phase !== 'LOADING') {
    if (gameState.phase !== 'FLIPPING') {
      updateCarousel(); // 更新轮播位置

      // 处理滚动逻辑
      if (gameState.phase === 'IDLE' || gameState.phase === 'SCROLLING') {
        if (gameState.isHandOpen || gameState.isDragging) {
          // 根据交互方式选择滚动步长
          const speed = gameState.mode === 'HAND' ? gameState.currentSpeed : gameState.scrollSpeed;
          gameState.scrollOffset += speed;
          gameState.phase = 'SCROLLING';

          if (gameState.mode === 'HAND') {
            const openPercent = Math.round(gameState.openness * 100);
            statusText.value = `状态: 命运之轮转动中... (${openPercent}%) 握拳抽取!`;
          } else {
            statusText.value = "状态: 命运之轮转动中... 握拳抽取!";
          }
        } else {
          gameState.phase = 'IDLE';
        }

        // 握拳瞬间执行选中卡牌 (wasFist 用于检测“下降沿/上升沿”触发)
        if (gameState.isFist && !gameState.wasFist) selectCard();

      } else if (gameState.phase === 'SHOWING') {
        // 【关键交互逻辑】在手势模式下，只要松开拳头就会触发卡牌消失（升华）
        if (gameState.mode === 'HAND' && !gameState.isFist && gameState.isHoldingFist) {
          dismissCard();
          gameState.isHoldingFist = false;
        }
      }
    }
    gameState.wasFist = gameState.isFist; // 记录上一帧状态，用于边缘触发判断
  }

  // 3. 执行最终渲染
  renderer.render(scene, camera);
};

// --- 交互具体方法 ---

/**
 * 选中卡牌并开始翻转动画
 */
const selectCard = () => {
  const center = getCenterCard();
  if (!center) return;

  gameState.phase = 'FLIPPING';
  gameState.selectedMesh = center.mesh;
  gameState.selectedData = center.data;
  gameState.selectedIndex = center.index;

  const mesh = center.mesh;
  const isRev = center.data.isReversed;

  let flipProgress = 0;
  const startRotY = mesh.rotation.y;
  const startPos = mesh.position.clone();
  const startScale = mesh.scale.x;
  const startRotZ = mesh.rotation.z;
  const targetRotZ = isRev ? Math.PI : 0;

  // 翻牌动画逻辑
  const flipAnimation = () => {
    flipProgress += 0.03;
    const ease = 1 - Math.pow(1 - Math.min(flipProgress, 1), 3); // 缓动

    // 旋转到正面
    mesh.rotation.y = THREE.MathUtils.lerp(startRotY, 0, ease);
    mesh.rotation.z = THREE.MathUtils.lerp(startRotZ, targetRotZ, ease);
    // 移动到正中心并靠近相机
    mesh.position.x = THREE.MathUtils.lerp(startPos.x, 0, ease);
    mesh.position.y = THREE.MathUtils.lerp(startPos.y, 0.5, ease);
    mesh.position.z = THREE.MathUtils.lerp(startPos.z, 4, ease);
    // 稍微放大增强冲击感
    mesh.scale.setScalar(THREE.MathUtils.lerp(startScale, 1.2, ease));

    if (flipProgress < 1) {
      requestAnimationFrame(flipAnimation);
    } else {
      gameState.phase = 'SHOWING';
      showResult(); // 动画结束，显示文字描述
    }
  };
  flipAnimation();
  gameState.isHoldingFist = true; // 标记正在持有握拳状态，等待释放触发升华
  statusText.value = "状态: 翻开命运之牌...";
};

/**
 * 展示抽卡结果
 */
const showResult = () => {
  const data = gameState.selectedData;
  const isRev = data.isReversed;
  const titleStr = data.name + (isRev ? " (逆位)" : " (正位)");
  const meaningStr = isRev ? data.r : data.u;

  // 更新响应式状态，触发 UI 层的渲染
  result.show = true;
  result.title = titleStr;
  result.meaning = meaningStr;
  result.isReversed = isRev;
  result.revealName = data.name;
  result.showNameReveal = true;

  // 加入历史记录
  historyList.value.unshift({ title: titleStr, meaning: meaningStr });
  statusText.value = "状态: 命运已揭晓! 松开拳头继续...";
};

/**
 * 撤销卡牌显示，触发粒子升华动画
 */
const dismissCard = () => {
  if (!gameState.selectedMesh) return;
  const mesh = gameState.selectedMesh;

  // 1. UI 动画控制
  result.showNameReveal = false; // 先关闭入场大字
  result.showSublimation = true; // 开启升华大字动画 (CSS 动画)

  // 2. 3D 特效触发
  starlights.push(new StarlightEffect(mesh.position.clone()));

  // 【修复卡牌残留】
  // 我们不再只是移除一张牌，而是决定“清理整个桌面”。
  // 这样做可以确保没有“旧卡牌留着表面”的问题。
  if (cardGroup) {
    cardGroup.traverse((child) => {
      if (child.isMesh) {
        // 如果是正在选中的这张，让它在 3D 空间消失（粒子接管）
        // 如果是其他残留牌，直接销毁
        child.visible = false;
      }
    });
  }

  // 清理引用
  gameState.cardMeshes[gameState.selectedIndex] = null;
  gameState.selectedMesh = null;
  gameState.selectedData = null;
  result.show = false;

  statusText.value = "状态: 灵感升华，准备下一轮...";

  // 2.5 秒后（等动画结束），彻底清空并重新生成
  setTimeout(() => {
    result.showSublimation = false;
    spawnAllCards(); // 彻底清理 cardGroup 并重新洗牌生成
  }, 2500);
};

const toggleMode = () => {
  gameState.mode = gameState.mode === 'HAND' ? 'MOUSE' : 'HAND';
  resetUIForNewRound();
};

const toggleImmersive = () => {
  isImmersive.value = !isImmersive.value;
};

// --- MediaPipe 手势识别逻辑 ---
let hands, cameraPipe;

/**
 * MediaPipe 识别结果回调
 */
const onResults = (results) => {
  if (!outputCanvas.value) return;

  // 1. 确保预览画布尺寸同步
  if (outputCanvas.value.width !== results.image.width) {
    outputCanvas.value.width = results.image.width;
    outputCanvas.value.height = results.image.height;
  }

  const canvasCtx = outputCanvas.value.getContext('2d');
  canvasCtx.save();
  canvasCtx.clearRect(0, 0, outputCanvas.value.width, outputCanvas.value.height);
  // 绘制摄像头原图
  canvasCtx.drawImage(results.image, 0, 0, outputCanvas.value.width, outputCanvas.value.height);

  // 2. 如果检测到手部关键点
  if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
    const landmarks = results.multiHandLandmarks[0];

    // 绘制手部骨架（使用 MediaPipe 提供的辅助函数）
    window.drawConnectors(canvasCtx, landmarks, window.HAND_CONNECTIONS, { color: '#00FF00', lineWidth: 2 });
    window.drawLandmarks(canvasCtx, landmarks, { color: '#FF0000', lineWidth: 1, radius: 1.5 });

    const indexTip = landmarks[8];
    const wrist = landmarks[0];

    // 【算法解释：手势识别】
    // 我们通过计算 5 个指尖到手腕的平均欧式距离 (avgDistToWrist) 来判断手势。
    // - 距离大 = 手掌张开
    // - 距离小 = 握拳
    const tips = [landmarks[4], landmarks[8], landmarks[12], landmarks[16], landmarks[20]];
    let avgDistToWrist = 0;
    tips.forEach(tip => {
      avgDistToWrist += Math.sqrt(Math.pow(tip.x - wrist.x, 2) + Math.pow(tip.y - wrist.y, 2));
    });
    avgDistToWrist /= 5;

    // 定义检测阈值
    const rawFist = avgDistToWrist < 0.22;
    const rawOpen = avgDistToWrist > 0.28;

    // 计算“张开程度”，用于控制滚动速度（张得越开转得越快）
    if (rawOpen) {
      const minOpen = 0.28, maxOpen = 0.36;
      let openness = (avgDistToWrist - minOpen) / (maxOpen - minOpen);
      openness = Math.max(0, Math.min(1, openness));
      gameState.openness = openness;

      const minSpeed = 0.03, maxSpeed = 0.15;
      gameState.currentSpeed = minSpeed + openness * (maxSpeed - minSpeed);
    } else {
      gameState.currentSpeed = 0;
      gameState.openness = 0;
    }

    // 防抖处理：必须连续 N 帧检测到该手势，才认为状态改变。
    // 这样可以解决摄像头画面噪点导致的“闪烁”问题。
    if (rawFist) {
      gameState.fistCounter++;
      gameState.openCounter = 0;
    } else if (rawOpen) {
      gameState.openCounter++;
      gameState.fistCounter = 0;
    } else {
      gameState.fistCounter = Math.max(0, gameState.fistCounter - 1);
      gameState.openCounter = Math.max(0, gameState.openCounter - 1);
    }

    // 状态确认
    gameState.isFist = gameState.fistCounter >= gameState.gestureThreshold;
    gameState.isHandOpen = gameState.openCounter >= gameState.gestureThreshold;

    gestureLabel.value = gameState.isFist ? "✊ 握拳" : (gameState.isHandOpen ? "🖐 张开" : "等待手势...");

    // 更新屏幕上的虚拟准星（基于食指指尖）
    if (cursorDebug.value) {
      cursorDebug.value.style.display = 'block';
      // 注意：MediaPipe 坐标是 0-1 的归一化值，且 x 是镜像的
      cursorDebug.value.style.left = `${(1 - indexTip.x) * window.innerWidth}px`;
      cursorDebug.value.style.top = `${indexTip.y * window.innerHeight}px`;
    }
  } else {
    // 未检测到手部，重置所有状态
    gestureLabel.value = "未检测";
    gameState.fistCounter = 0;
    gameState.openCounter = 0;
    gameState.isHandOpen = false;
    gameState.isFist = false;
    if (cursorDebug.value) cursorDebug.value.style.display = 'none';
  }
  canvasCtx.restore();
};

/**
 * 初始化 MediaPipe Hands
 */
const initMediaPipe = () => {
  hands = new window.Hands({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
  });
  hands.setOptions({
    maxNumHands: 1,
    modelComplexity: 1,
    minDetectionConfidence: 0.5,
    minTrackingConfidence: 0.5
  });
  hands.onResults(onResults);

  cameraPipe = new window.Camera(inputVideo.value, {
    onFrame: async () => {
      await hands.send({ image: inputVideo.value });
    },
    width: 640,
    height: 480
  });
  cameraPipe.start();
};

// --- Vue 生命周期钩子 ---

onMounted(() => {
  initThree();     // 1. 启动 3D 引擎
  initMediaPipe(); // 2. 启动手势识别
  spawnAllCards(); // 3. 初始生成卡牌
  animate();       // 4. 开启渲染循环

  // 注册全局事件
  window.addEventListener('keydown', (e) => {
    if (e.key.toLowerCase() === 'i') toggleImmersive();
    if (e.key.toLowerCase() === 'm') toggleMode();
  });

  // 鼠标交互绑定
  window.addEventListener('mousedown', () => { if (gameState.mode === 'MOUSE') gameState.isDragging = true; });
  window.addEventListener('mouseup', () => { if (gameState.mode === 'MOUSE') gameState.isDragging = false; });
  window.addEventListener('dblclick', () => {
    if (gameState.mode === 'MOUSE') {
      if (gameState.phase === 'IDLE' || gameState.phase === 'SCROLLING') selectCard();
      else if (gameState.phase === 'SHOWING') dismissCard();
    }
  });
});

onUnmounted(() => {
  // 必须清理掉动画和摄像头，否则组件销毁后依然会占用资源
  cancelAnimationFrame(animationId);
  if (cameraPipe) cameraPipe.stop();
  if (renderer) renderer.dispose();
  window.removeEventListener('resize', onWindowResize);
});
</script>

<style scoped>
/* 
  全屏 3D 容器
  z-index: 1 确保它在最底层
*/
#canvas-container {
  width: 100vw;
  height: 100vh;
  position: absolute;
  top: 0;
  left: 0;
  z-index: 1;
  background-color: #050510;
}

/* 
  UI 层：负责文字、按钮和历史记录
  pointer-events: none 允许点击穿透到下层的 3D 场景（除非是特定按钮）
*/
#ui-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 10;
  pointer-events: none;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  color: #eee;
  font-family: "Microsoft YaHei", sans-serif;
}

.top-bar {
  padding: 20px;
  background: linear-gradient(to bottom, rgba(0, 0, 0, 0.9), transparent);
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  pointer-events: auto;
}

.status-box {
  background: rgba(0, 0, 0, 0.6);
  padding: 15px;
  border: 1px solid #444;
  border-radius: 8px;
  backdrop-filter: blur(5px);
  min-width: 200px;
}

h1 {
  margin: 0 0 10px 0;
  font-size: 1.2rem;
  color: #d4af37;
  text-shadow: 0 0 10px #d4af37;
}

.status-text {
  color: #fff;
  font-size: 0.9rem;
  margin-bottom: 5px;
}

.mode-switch {
  cursor: pointer;
  padding: 5px 12px;
  background: #333;
  border: 1px solid #666;
  color: white;
  border-radius: 4px;
  font-size: 0.8rem;
  transition: 0.2s;
}

.mode-switch:hover {
  background: #d4af37;
  color: #000;
}

/* 沉浸模式样式切换 */
.immersive #ui-layer {
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.5s;
}

.immersive #cursor-debug {
  display: none !important;
}

.immersive #loader {
  display: none !important;
}

#history-panel {
  width: 260px;
  max-height: 40vh;
  overflow-y: auto;
  background: rgba(0, 0, 0, 0.7);
  border: 1px solid #333;
  border-radius: 8px;
  padding: 10px;
  pointer-events: auto;
  font-size: 0.9rem;
}

/* 自定义滚动条样式，让其更符合魔法氛围 */
#history-panel::-webkit-scrollbar {
  width: 6px;
}

#history-panel::-webkit-scrollbar-thumb {
  background: #444;
  border-radius: 3px;
}

.history-item {
  margin-bottom: 8px;
  border-bottom: 1px solid #444;
  padding-bottom: 5px;
  animation: fadeIn 0.5s ease;
}

.history-name {
  color: #d4af37;
  font-weight: bold;
}

.history-meaning {
  font-size: 0.8rem;
  color: #aaa;
  margin-top: 2px;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateX(20px);
  }

  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.bottom-hint {
  text-align: center;
  padding: 20px;
  text-shadow: 0 0 5px black;
  font-size: 1.1rem;
  opacity: 0.8;
  background: linear-gradient(to top, rgba(0, 0, 0, 0.8), transparent);
}

/* 加载动画相关 */
#loader {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: #000;
  z-index: 100;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 5px solid #333;
  border-top: 5px solid #d4af37;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(360deg);
  }
}

/* 结果浮层样式 */
#result-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  transition: opacity 0.5s;
  pointer-events: none;
  width: 80%;
}

.result-title {
  font-size: 3rem;
  text-shadow: 0 0 30px rgba(212, 175, 55, 0.5);
  margin: 0;
}

.result-sub {
  font-size: 1.4rem;
  color: #fff;
  margin-top: 15px;
  text-shadow: 0 0 10px #000;
  line-height: 1.4;
}

/* 揭示动画大字 */
#card-name-reveal {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 4rem;
  font-weight: bold;
  color: transparent;
  background: linear-gradient(135deg, #d4af37 0%, #f4e4ba 25%, #d4af37 50%, #aa8c2c 75%, #d4af37 100%);
  background-size: 200% 200%;
  -webkit-background-clip: text;
  background-clip: text;
  text-shadow: 0 0 40px rgba(212, 175, 55, 0.8), 0 0 80px rgba(212, 175, 55, 0.4);
  opacity: 0;
  pointer-events: none;
  z-index: 50;
  letter-spacing: 8px;
}

/* show 状态：升华消失动画 */
#card-name-reveal.show {
  animation: revealName 2.5s ease-out forwards, shimmer 2s linear infinite;
}

@keyframes revealName {
  0% {
    opacity: 0;
    transform: translate(-50%, -50%) scale(0.5);
    filter: blur(20px);
  }

  30% {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1.1);
    filter: blur(0);
  }

  50% {
    transform: translate(-50%, -50%) scale(1);
  }

  80% {
    opacity: 1;
  }

  100% {
    opacity: 0;
    transform: translate(-50%, -50%) scale(1.2);
  }
}

/* showing 状态：入场展示动画 */
#card-name-reveal.showing {
  animation: revealIn 0.8s ease-out forwards, shimmer 2s linear infinite;
}

@keyframes revealIn {
  0% {
    opacity: 0;
    transform: translate(-50%, -80%) scale(0.5);
    filter: blur(10px);
  }

  100% {
    opacity: 1;
    transform: translate(-50%, -80%) scale(1);
    filter: blur(0);
  }
}

@keyframes shimmer {
  0% {
    background-position: 200% 0;
  }

  100% {
    background-position: -200% 0;
  }
}

/* 虚拟准星 */
#cursor-debug {
  position: absolute;
  width: 24px;
  height: 24px;
  border: 2px solid rgba(255, 255, 255, 0.6);
  border-radius: 50%;
  transform: translate(-50%, -50%);
  pointer-events: none;
  z-index: 20;
  display: none;
  box-shadow: 0 0 10px rgba(255, 255, 255, 0.2);
}

/* 摄像头预览窗口 */
#camera-preview {
  position: absolute;
  bottom: 20px;
  right: 20px;
  width: 320px;
  height: 240px;
  border: 2px solid #d4af37;
  border-radius: 8px;
  overflow: hidden;
  z-index: 90;
  background: #000;
  box-shadow: 0 0 15px rgba(0, 0, 0, 0.8);
}

#output-canvas {
  width: 100%;
  height: 100%;
  transform: scaleX(-1);
  /* 镜像翻转，符合镜子习惯 */
}

#gesture-label {
  position: absolute;
  bottom: 10px;
  left: 10px;
  color: #0f0;
  font-size: 16px;
  font-weight: bold;
  text-shadow: 2px 2px 4px black;
  z-index: 10;
  background: rgba(0, 0, 0, 0.5);
  padding: 4px 8px;
  border-radius: 4px;
}

#input-video {
  display: none;
}
</style>
