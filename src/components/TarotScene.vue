<template>
  <div :class="{ immersive: isImmersive }">
    <!-- 加载遮罩 -->
    <div v-if="isLoading" id="loader">
      <div class="spinner"></div>
      <p style="margin-top:20px; color:#aaa;">正在连接命运之线...</p>
    </div>

    <div ref="canvasContainer" id="canvas-container"></div>
    <div ref="cursorDebug" id="cursor-debug"></div>

    <div id="ui-layer">
      <div class="top-bar">
        <div class="status-box">
          <h1>🔮 沉浸式塔罗</h1>
          <div class="status-text">{{ statusText }}</div>
          <button @click="toggleMode" class="mode-switch">切换输入模式</button>
          <button @click="toggleImmersive" class="mode-switch" style="margin-left:5px;">沉浸模式 (I)</button>
        </div>
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

      <div id="result-overlay" :style="{ opacity: result.show ? 1 : 0 }">
        <h2 class="result-title" :style="{ color: result.isReversed ? '#aaa' : '#d4af37' }">{{ result.title }}</h2>
        <div class="result-sub">{{ result.meaning }}</div>
      </div>

      <div class="bottom-hint">
        {{ guideText }}
      </div>
    </div>

    <div ref="cardNameReveal" id="card-name-reveal"
      :class="{ showing: result.showNameReveal, show: result.showSublimation }">
      {{ result.revealName }}
    </div>

    <div id="camera-preview">
      <canvas ref="outputCanvas" id="output-canvas"></canvas>
      <div id="gesture-label">{{ gestureLabel }}</div>
    </div>

    <video ref="inputVideo" id="input-video"></video>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';

// --- 数据配置 ---
const TAROT_DATA = [
  { name: "0 愚人", u: "新的开始，冒险，天真", r: "鲁莽，轻率，风险", imgId: 0 },
  { name: "I 魔术师", u: "创造力，技能，意志力", r: "欺骗，沟通不畅", imgId: 1 },
  { name: "II 女祭司", u: "直觉，潜意识，神秘", r: "压抑，情绪不稳定", imgId: 2 },
  { name: "III 皇后", u: "丰饶，母性，自然", r: "依赖，创造力受阻", imgId: 3 },
  { name: "IV 皇帝", u: "权威，结构，控制", r: "暴政，僵化，冷酷", imgId: 4 },
  { name: "V 教皇", u: "传统，信仰，指引", r: "反叛，虚伪，限制", imgId: 5 },
  { name: "VI 恋人", u: "爱，和谐，选择", r: "不平衡，分离", imgId: 6 },
  { name: "VII 战车", u: "胜利，意志，控制", r: "失控，攻击性", imgId: 7 },
  { name: "VIII 力量", u: "勇气，耐心，同情", r: "软弱，自我怀疑", imgId: 8 },
  { name: "IX 隐士", u: "内省，孤独，指引", r: "孤立，迷失", imgId: 9 },
  { name: "X 命运之轮", u: "改变，周期，命运", r: "坏运气，抵抗改变", imgId: 10 },
  { name: "XI 正义", u: "公平，真理，法律", r: "不公，偏见", imgId: 11 },
  { name: "XII 倒吊人", u: "牺牲，新视角，等待", r: "无谓牺牲，拖延", imgId: 12 },
  { name: "XIII 死神", u: "结束，转变，重生", r: "停滞，恐惧改变", imgId: 13 },
  { name: "XIV 节制", u: "平衡，适度，耐心", r: "失衡，极端", imgId: 14 },
  { name: "XV 恶魔", u: "束缚，物质主义", r: "释放，打破枷锁", imgId: 15 },
  { name: "XVI 高塔", u: "突变，灾难，启示", r: "避免灾难，恐惧", imgId: 16 },
  { name: "XVII 星星", u: "希望，灵感，平静", r: "绝望，缺乏信心", imgId: 17 },
  { name: "XVIII 月亮", u: "幻觉，恐惧，潜意识", r: "释放恐惧，清晰", imgId: 18 },
  { name: "XIX 太阳", u: "快乐，成功，活力", r: "悲伤，暂时受阻", imgId: 19 },
  { name: "XX 审判", u: "复活，觉醒，宽恕", r: "自我怀疑，拒绝改变", imgId: 20 },
  { name: "XXI 世界", u: "完成，整合，旅行", r: "未完成，缺乏闭环", imgId: 21 }
];

const IMG_BASE_URL = "https://raw.githubusercontent.com/dowsonmicic/Tarot_butterfly/refs/heads/images/docs/";
const RWS_MAP = [
  "1.jpg", "2.jpg", "3.jpg", "4.jpg", "5.jpg", "6.jpg",
  "7.jpg", "8.jpg", "9.jpg", "10.jpg", "11.jpg", "12.jpg",
  "13.jpg", "14.jpg", "15.jpg", "16.jpg", "17.jpg", "18.jpg",
  "19.jpg", "20.jpg", "21.jpg", "22.jpg"
];
const BACK_IMG_URL = IMG_BASE_URL + "bm.jpg";

// --- 响应式状态 ---
const isLoading = ref(true);
const statusText = ref("初始化中...");
const guideText = ref("初始化手势识别中...");
const historyList = ref([]);
const isImmersive = ref(false);
const gestureLabel = ref("未检测");
const result = reactive({
  show: false,
  title: "",
  meaning: "",
  isReversed: false,
  revealName: "",
  showNameReveal: false,
  showSublimation: false
});

// --- DOM 引用 ---
const canvasContainer = ref(null);
const cursorDebug = ref(null);
const outputCanvas = ref(null);
const inputVideo = ref(null);

// --- Three.js 变量 ---
let scene, camera, renderer, starfield, textureLoader, backTexture;
let starlights = [];
let animationId;
let cardGroup; // 用于统一管理卡牌，防止泄露
let currentLoadId = 0; // 用于防止异步加载冲突

const gameState = reactive({
  mode: 'HAND',
  cardMeshes: [],
  cardDataList: [],
  scrollOffset: 0,
  scrollSpeed: 0.07,
  currentSpeed: 0.10,
  phase: 'IDLE',
  selectedMesh: null,
  selectedData: null,
  selectedIndex: -1,
  cursor: { x: 0, y: 0 },
  isHandOpen: false,
  openness: 0,
  isFist: false,
  wasFist: false,
  isDragging: false,
  fistCounter: 0,
  openCounter: 0,
  gestureThreshold: 5,
  isHoldingFist: false,
  fistHoldStartTime: 0,
  fistHoldDuration: 2000
});

const CARD_COUNT = 22;
const CARD_SPACING = 3.2;
const TOTAL_WIDTH = CARD_COUNT * CARD_SPACING;

// --- 星空背景系统 ---
class Starfield {
  constructor() {
    this.count = 2000;
    this.geometry = new THREE.BufferGeometry();
    const positions = new Float32Array(this.count * 3);
    const colors = new Float32Array(this.count * 3);
    const sizes = new Float32Array(this.count);

    for (let i = 0; i < this.count; i++) {
      positions[i * 3] = (Math.random() - 0.5) * 100;
      positions[i * 3 + 1] = (Math.random() - 0.5) * 100;
      positions[i * 3 + 2] = (Math.random() - 0.5) * 50 - 20;

      const r = 0.8 + Math.random() * 0.2;
      const g = 0.8 + Math.random() * 0.2;
      const b = 0.9 + Math.random() * 0.1;
      colors[i * 3] = r;
      colors[i * 3 + 1] = g;
      colors[i * 3 + 2] = b;

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
      blending: THREE.AdditiveBlending,
      sizeAttenuation: true
    });

    this.mesh = new THREE.Points(this.geometry, this.material);
    scene.add(this.mesh);
  }

  update() {
    const positions = this.geometry.attributes.position.array;
    const time = Date.now() * 0.0005;

    for (let i = 0; i < this.count; i++) {
      positions[i * 3 + 2] += 0.01;
      if (positions[i * 3 + 2] > 20) {
        positions[i * 3 + 2] = -30;
      }
    }
    this.geometry.attributes.position.needsUpdate = true;
    this.material.opacity = 0.6 + Math.sin(time) * 0.2;
  }
}

// --- 星光升华粒子系统 ---
class StarlightEffect {
  constructor(position) {
    this.particlesCount = 2500;
    this.geometry = new THREE.BufferGeometry();
    const positions = [];
    const velocities = [];
    const colors = [];
    const life = [];

    const w = 4.0, h = 5.5;
    for (let i = 0; i < this.particlesCount; i++) {
      positions.push(
        position.x + (Math.random() - 0.5) * w,
        position.y + (Math.random() - 0.5) * h,
        position.z + (Math.random() - 0.5) * 0.2
      );

      const speed = 0.02 + Math.random() * 0.05;
      velocities.push(
        (Math.random() - 0.5) * 0.02,
        speed * 1.5 + 0.01,
        (Math.random() - 0.5) * 0.02
      );

      const colorChoice = Math.random();
      if (colorChoice < 0.5) {
        colors.push(1.0, 0.9, 0.5);
      } else if (colorChoice < 0.8) {
        colors.push(1.0, 1.0, 1.0);
      } else {
        colors.push(0.9, 0.7, 1.0);
      }

      life.push(1.0 + Math.random() * 1.2);
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
        positions[i * 3] += velocities[i * 3];
        positions[i * 3 + 1] += velocities[i * 3 + 1];
        positions[i * 3 + 2] += velocities[i * 3 + 2];

        const wave = elapsed * 2 + i * 0.05;
        positions[i * 3] += Math.sin(wave) * 0.005;
        positions[i * 3 + 2] += Math.cos(wave) * 0.005;

        lives[i] -= 0.004;
      }
    }

    this.material.opacity = Math.max(0, this.material.opacity - 0.004);
    this.geometry.attributes.position.needsUpdate = true;

    if (!alive || this.material.opacity <= 0) {
      scene.remove(this.mesh);
      this.geometry.dispose();
      this.material.dispose();
      return false;
    }
    return true;
  }
}

// --- 初始化 Three.js ---
const initThree = () => {
  scene = new THREE.Scene();
  scene.fog = new THREE.FogExp2(0x050510, 0.03);

  camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.set(0, 0, 8);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  canvasContainer.value.appendChild(renderer.domElement);

  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.add(ambientLight);
  const dirLight = new THREE.DirectionalLight(0xffd700, 1.0);
  dirLight.position.set(5, 10, 7);
  scene.add(dirLight);
  const pointLight = new THREE.PointLight(0xaa88ff, 0.8);
  pointLight.position.set(-5, 2, 3);
  scene.add(pointLight);

  starfield = new Starfield();
  textureLoader = new THREE.TextureLoader();
  textureLoader.crossOrigin = 'anonymous';
  backTexture = textureLoader.load(BACK_IMG_URL);

  // 初始化卡牌容器
  cardGroup = new THREE.Group();
  scene.add(cardGroup);

  window.addEventListener('resize', onWindowResize);
};

const onWindowResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
};

// --- 卡牌逻辑 ---
const spawnAllCards = () => {
  gameState.phase = 'LOADING';
  statusText.value = "状态: 正在召唤命运之牌...";
  currentLoadId++; // 增加加载ID
  const thisLoadId = currentLoadId;

  // 彻底清理旧牌
  if (cardGroup) {
    // 递归释放资源
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
    cardGroup.clear();
  }

  gameState.cardMeshes = [];
  gameState.cardDataList = [];
  gameState.scrollOffset = 0;
  gameState.selectedMesh = null;
  gameState.selectedData = null;

  const shuffledIndices = Array.from({ length: 22 }, (_, i) => i);
  for (let i = shuffledIndices.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [shuffledIndices[i], shuffledIndices[j]] = [shuffledIndices[j], shuffledIndices[i]];
  }

  let loadedCount = 0;
  shuffledIndices.forEach((cardId, index) => {
    const cardData = TAROT_DATA[cardId];
    const isReversed = Math.random() < 0.5;
    const imgUrl = IMG_BASE_URL + RWS_MAP[cardData.imgId];

    gameState.cardDataList[index] = { ...cardData, isReversed };

    textureLoader.load(
      imgUrl,
      (tex) => {
        // 检查加载ID是否匹配，防止旧的异步请求在清理后又添加物体
        if (thisLoadId !== currentLoadId) {
          tex.dispose();
          return;
        }
        createCarouselCard(tex, index, isReversed);
        loadedCount++;
        if (loadedCount === CARD_COUNT) onAllCardsLoaded(thisLoadId);
      },
      undefined,
      () => {
        if (thisLoadId !== currentLoadId) return;
        const placeholder = new THREE.CanvasTexture(createPlaceholderCanvas(cardData.name));
        createCarouselCard(placeholder, index, isReversed);
        loadedCount++;
        if (loadedCount === CARD_COUNT) onAllCardsLoaded(thisLoadId);
      }
    );
  });
};

const createCarouselCard = (frontTex, index, isReversed) => {
  const geometry = new THREE.BoxGeometry(2.5, 4.0, 0.05);
  frontTex.colorSpace = THREE.SRGBColorSpace;
  backTexture.colorSpace = THREE.SRGBColorSpace;

  const sideMat = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.5 });
  const frontMat = new THREE.MeshStandardMaterial({ map: frontTex, roughness: 0.4, metalness: 0.1 });
  const backMat = new THREE.MeshStandardMaterial({ map: backTexture, roughness: 0.6 });

  const materials = [sideMat, sideMat, sideMat, sideMat, frontMat, backMat];
  const mesh = new THREE.Mesh(geometry, materials);
  mesh.userData = { index, isReversed, originalIndex: index, frontTex };
  mesh.position.y = -10;
  mesh.position.z = -2;
  mesh.rotation.y = Math.PI;

  cardGroup.add(mesh);
  gameState.cardMeshes[index] = mesh;
};

const onAllCardsLoaded = (loadId) => {
  if (loadId !== currentLoadId) return; // 再次检查

  let progress = 0;
  const intro = () => {
    if (loadId !== currentLoadId) return; // 动画过程中也要检查

    progress += 0.025;
    const easeOut = 1 - Math.pow(1 - Math.min(progress, 1), 3);

    gameState.cardMeshes.forEach((mesh, i) => {
      if (!mesh) return;
      const targetX = (i - CARD_COUNT / 2) * CARD_SPACING;
      mesh.position.x = targetX;
      mesh.position.y = THREE.MathUtils.lerp(-10, 0, easeOut);
      mesh.position.z = -2;
      mesh.rotation.y = Math.PI; // 初始背面
      if (mesh.userData.isReversed) mesh.rotation.z = Math.PI;
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

const resetUIForNewRound = () => {
  result.show = false;
  statusText.value = gameState.mode === 'HAND'
    ? "状态: 张开手掌开始滚动"
    : "状态: 按住左键开始滚动";
  guideText.value = gameState.mode === 'HAND'
    ? "🖐 张开=滚动 | ✊ 握拳=抽取 | 松开=升华"
    : "🖱 按住=滚动 | 双击=抽取 | 再次双击=升华";
};

const updateCarousel = () => {
  gameState.cardMeshes.forEach((mesh, i) => {
    if (!mesh || mesh === gameState.selectedMesh) return;
    let x = (i - CARD_COUNT / 2) * CARD_SPACING + gameState.scrollOffset;
    while (x > TOTAL_WIDTH / 2) x -= TOTAL_WIDTH;
    while (x < -TOTAL_WIDTH / 2) x += TOTAL_WIDTH;

    mesh.position.x = x;
    const distFromCenter = Math.abs(x);
    const scale = THREE.MathUtils.lerp(1.15, 0.9, Math.min(distFromCenter / 8, 1));
    const zOffset = THREE.MathUtils.lerp(1, 0, Math.min(distFromCenter / 8, 1));

    mesh.scale.lerp(new THREE.Vector3(scale, scale, scale), 0.15);
    mesh.position.z = THREE.MathUtils.lerp(mesh.position.z, -2 + zOffset, 0.15);
    mesh.position.y = THREE.MathUtils.lerp(mesh.position.y, 0, 0.1);

    // 增加一点旋转效果使卡牌看起来更立体
    mesh.rotation.y = Math.PI + x * 0.05;
  });
};

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

// --- 核心更新循环 ---
const animate = () => {
  animationId = requestAnimationFrame(animate);
  if (starfield) starfield.update();
  starlights = starlights.filter(s => s.update());

  if (gameState.cardMeshes.length > 0 && gameState.phase !== 'LOADING') {
    if (gameState.phase !== 'FLIPPING') {
      updateCarousel();
      if (gameState.phase === 'IDLE' || gameState.phase === 'SCROLLING') {
        if (gameState.isHandOpen || gameState.isDragging) {
          // 使用动态计算的速度
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
        if (gameState.isFist && !gameState.wasFist) selectCard();
      } else if (gameState.phase === 'SHOWING') {
        // 如果处于显示状态，且是手势模式，且之前是握拳选择的，那么当手松开时消失
        if (gameState.mode === 'HAND' && !gameState.isFist && gameState.isHoldingFist) {
          dismissCard();
          gameState.isHoldingFist = false;
        }
      }
    }
    gameState.wasFist = gameState.isFist;
  }

  renderer.render(scene, camera);
};

// --- 交互方法 ---
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

  const flipAnimation = () => {
    flipProgress += 0.03;
    const ease = 1 - Math.pow(1 - Math.min(flipProgress, 1), 3);
    mesh.rotation.y = THREE.MathUtils.lerp(startRotY, 0, ease);
    mesh.rotation.z = THREE.MathUtils.lerp(startRotZ, targetRotZ, ease);
    mesh.position.x = THREE.MathUtils.lerp(startPos.x, 0, ease);
    mesh.position.y = THREE.MathUtils.lerp(startPos.y, 0.5, ease);
    mesh.position.z = THREE.MathUtils.lerp(startPos.z, 4, ease);
    mesh.scale.setScalar(THREE.MathUtils.lerp(startScale, 1.2, ease));

    if (flipProgress < 1) {
      requestAnimationFrame(flipAnimation);
    } else {
      gameState.phase = 'SHOWING';
      showResult();
    }
  };
  flipAnimation();
  gameState.isHoldingFist = true; // 标记正在持有握拳状态，等待释放
  statusText.value = "状态: 翻开命运之牌...";
};

const showResult = () => {
  const data = gameState.selectedData;
  const isRev = data.isReversed;
  const titleStr = data.name + (isRev ? " (逆位)" : " (正位)");
  const meaningStr = isRev ? data.r : data.u;

  result.show = true;
  result.title = titleStr;
  result.meaning = meaningStr;
  result.isReversed = isRev;
  result.revealName = data.name;
  result.showNameReveal = true;

  historyList.value.unshift({ title: titleStr, meaning: meaningStr });
  statusText.value = "状态: 命运已揭晓! 松开拳头继续...";
};

const dismissCard = () => {
  if (!gameState.selectedMesh) return;
  const mesh = gameState.selectedMesh;

  // 显示牌名升华动画 (show 状态)
  result.showNameReveal = false; // 先关闭 showing
  result.showSublimation = true; // 开启 show 动画

  starlights.push(new StarlightEffect(mesh.position.clone()));
  cardGroup.remove(mesh);
  gameState.cardMeshes[gameState.selectedIndex] = null;
  gameState.selectedMesh = null;
  gameState.selectedData = null;
  result.show = false;

  statusText.value = "状态: 灵感升华，准备下一轮...";

  // 2.5秒后重新生成
  setTimeout(() => {
    result.showSublimation = false;
    spawnAllCards(); // 每次抽完都重新生成，保证场景干净
  }, 2500);
};

const toggleMode = () => {
  gameState.mode = gameState.mode === 'HAND' ? 'MOUSE' : 'HAND';
  resetUIForNewRound();
};

const toggleImmersive = () => {
  isImmersive.value = !isImmersive.value;
};

// --- MediaPipe 手势识别 ---
let hands, cameraPipe;

const onResults = (results) => {
  if (!outputCanvas.value) return;

  // 确保画布尺寸与输入图像一致
  if (outputCanvas.value.width !== results.image.width) {
    outputCanvas.value.width = results.image.width;
    outputCanvas.value.height = results.image.height;
  }

  const canvasCtx = outputCanvas.value.getContext('2d');
  canvasCtx.save();
  canvasCtx.clearRect(0, 0, outputCanvas.value.width, outputCanvas.value.height);
  canvasCtx.drawImage(results.image, 0, 0, outputCanvas.value.width, outputCanvas.value.height);

  if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
    const landmarks = results.multiHandLandmarks[0];
    window.drawConnectors(canvasCtx, landmarks, window.HAND_CONNECTIONS, { color: '#00FF00', lineWidth: 2 });
    window.drawLandmarks(canvasCtx, landmarks, { color: '#FF0000', lineWidth: 1, radius: 1.5 });

    const indexTip = landmarks[8];
    const thumbTip = landmarks[4];
    const middleTip = landmarks[12];
    const ringTip = landmarks[16];
    const pinkyTip = landmarks[20];
    const wrist = landmarks[0];

    // 计算所有指尖到手腕的平均距离
    const tips = [landmarks[4], landmarks[8], landmarks[12], landmarks[16], landmarks[20]];
    let avgDistToWrist = 0;
    tips.forEach(tip => {
      avgDistToWrist += Math.sqrt(Math.pow(tip.x - wrist.x, 2) + Math.pow(tip.y - wrist.y, 2));
    });
    avgDistToWrist /= 5;

    // 原始检测阈值
    const rawFist = avgDistToWrist < 0.22;
    const rawOpen = avgDistToWrist > 0.28;

    if (rawOpen) {
      const minOpen = 0.28;
      const maxOpen = 0.36;
      let openness = (avgDistToWrist - minOpen) / (maxOpen - minOpen);
      openness = Math.max(0, Math.min(1, openness));
      gameState.openness = openness;

      const minSpeed = 0.03;
      const maxSpeed = 0.15;
      gameState.currentSpeed = minSpeed + openness * (maxSpeed - minSpeed);
    } else {
      gameState.currentSpeed = 0;
      gameState.openness = 0;
    }

    // 防抖处理
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

    // 达到阈值才确认状态
    gameState.isFist = gameState.fistCounter >= gameState.gestureThreshold;
    gameState.isHandOpen = gameState.openCounter >= gameState.gestureThreshold;

    gestureLabel.value = gameState.isFist ? "✊ 握拳" : (gameState.isHandOpen ? "🖐 张开" : "等待手势...");

    // 更新虚拟准星
    if (cursorDebug.value) {
      cursorDebug.value.style.display = 'block';
      cursorDebug.value.style.left = `${(1 - indexTip.x) * window.innerWidth}px`;
      cursorDebug.value.style.top = `${indexTip.y * window.innerHeight}px`;
    }
  } else {
    gestureLabel.value = "未检测";
    gameState.fistCounter = 0;
    gameState.openCounter = 0;
    gameState.isHandOpen = false;
    gameState.isFist = false;
    if (cursorDebug.value) cursorDebug.value.style.display = 'none';
  }
  canvasCtx.restore();
};

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

onMounted(() => {
  initThree();
  initMediaPipe();
  spawnAllCards();
  animate();

  // 键盘快捷键
  window.addEventListener('keydown', (e) => {
    if (e.key.toLowerCase() === 'i') toggleImmersive();
    if (e.key.toLowerCase() === 'm') toggleMode();
  });

  // 鼠标交互
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
  cancelAnimationFrame(animationId);
  if (cameraPipe) cameraPipe.stop();
  if (renderer) renderer.dispose();
  window.removeEventListener('resize', onWindowResize);
});
</script>

<style scoped>
#canvas-container {
  width: 100vw;
  height: 100vh;
  position: absolute;
  top: 0;
  left: 0;
  z-index: 1;
  background-color: #050510;
}

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

/* 沉浸模式 */
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

.immersive #card-name-reveal {
  display: block !important;
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

/* 自定义滚动条 */
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

.immersive #ui-layer {
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.5s;
}
</style>
