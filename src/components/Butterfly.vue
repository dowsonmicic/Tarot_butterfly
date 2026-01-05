<template>
  <div ref="container" class="butterfly-container">
    <div class="loading-overlay" v-if="loading">
      <div class="spinner"></div>
      <p>正在加载蝴蝶模型... {{ progress }}%</p>
    </div>
  </div>
</template>

<script setup>
/**
 * Butterfly.vue
 * 这是一个基于 Three.js 的 GLTF 模型展示组件
 * 核心功能：
 * 1. 加载并展示蝴蝶 3D 模型
 * 2. 蝴蝶自动跟随光标飞行，带有平滑的位移和转向动画
 * 3. 当蝴蝶接近光标位置时，会自动切换为 45 度仰望姿态
 * 4. 包含响应式窗口调整和资源清理逻辑
 */
import { onMounted, onUnmounted, ref } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

// 使用 Vite 的 ?url 后缀来获取静态资源的正确路径，避免手动移动文件
import modelUrl from '../../resource/borboleta_azul_-_butterfly/scene.gltf?url';

// --- 响应式状态 ---
const container = ref(null);      // 渲染容器的引用
const loading = ref(true);        // 加载状态控制
const progress = ref(0);          // 加载进度百分比

// --- Three.js 核心变量 ---
let scene, camera, renderer, animationId, mixer;
let butterflyGroup;               // 蝴蝶的父容器，负责整体的位移、水平转向和随机晃动
let butterflyModel;               // 蝴蝶模型本身，负责初始朝向校正和局部仰角动作
let clock = new THREE.Clock();    // 用于动画混合器的时间追踪
let mouse = new THREE.Vector2();  // 存储鼠标的 NDC 坐标
let targetPosition = new THREE.Vector3(); // 蝴蝶飞行的目标点（3D 世界坐标）
let raycaster = new THREE.Raycaster();    // 射线检测工具

// --- 飞行交互配置 ---
const followConfig = {
  moveSpeed: 0.05,    // 位移插值系数：数值越大，跟随越紧凑，惯性越小
  turnSpeed: 0.05,    // 转向插值系数：数值越大，转弯越灵敏
  tiltSpeed: 0.05,    // 仰角插值系数：控制抬头/低头动作的响应速度
  tiltThreshold: 3.0  // 仰角触发距离：当蝴蝶距离目标点小于此值时开始抬头
};

/**
 * 初始化 Three.js 场景
 */
const init = () => {
  // 1. 场景初始化
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111122); // 深蓝色背景，营造夜晚感
  scene.fog = new THREE.Fog(0x111122, 5, 15);   // 线性雾，增加空间深度感

  // 2. 相机设置
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(0, 1, 5); // 初始相机位置

  // 3. 渲染器设置
  renderer = new THREE.WebGLRenderer({ antialias: true }); // 开启抗锯齿
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio); // 适配高分屏
  renderer.shadowMap.enabled = true;               // 开启阴影
  renderer.outputColorSpace = THREE.SRGBColorSpace; // 设置色彩空间
  container.value.appendChild(renderer.domElement);

  // 4. 光照系统
  const ambientLight = new THREE.AmbientLight(0xffffff, 1.5); // 基础环境光
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 2); // 模拟主光源
  directionalLight.position.set(5, 5, 5);
  directionalLight.castShadow = true; // 产生阴影
  scene.add(directionalLight);

  // 5. 模型加载逻辑
  const loader = new GLTFLoader();
  loader.load(
    modelUrl,
    (gltf) => {
      // 获取加载成功的模型场景对象
      butterflyModel = gltf.scene;

      // 【关键设计：父子容器解耦】
      // 创建 butterflyGroup 作为父容器，负责处理世界坐标系的位移(Translation)和水平转向(Yaw)。
      // butterflyModel 作为子对象，负责处理局部坐标系的校正(Orientation)和俯仰动作(Pitch)。
      // 这样做的好处是：在执行 lookAt 转向时，不会干扰到模型自身的抬头/低头动画。
      butterflyGroup = new THREE.Group();

      // --- 自动缩放与居中逻辑 ---
      // 1. 计算模型在原始状态下的包围盒
      const box = new THREE.Box3().setFromObject(butterflyModel);
      const center = box.getCenter(new THREE.Vector3());
      const size = box.getSize(new THREE.Vector3());

      // 2. 根据包围盒的最大维度，将模型统一缩放到约 0.8 个单位大小
      const maxDim = Math.max(size.x, size.y, size.z);
      const scale = 0.8 / maxDim;
      butterflyModel.scale.setScalar(scale);

      // 3. 将模型几何中心移动到本地坐标原点(0,0,0)，确保旋转时绕中心点旋转
      butterflyModel.position.sub(center.multiplyScalar(scale));

      // 【核心校正：朝向对齐】
      // 大多数外部 GLTF 模型的正前方(Forward)不一定是 +Z 轴。
      // 这里根据调试结果，将模型绕 Y 轴旋转 -90 度，使其头部指向 +Z 轴。
      // 只有头部指向 +Z，Three.js 的 lookAt 函数才能让蝴蝶“头朝前”飞行。
      butterflyModel.rotation.y = -Math.PI / 2;

      // 递归遍历模型的所有子部件，开启阴影投射与接收
      butterflyModel.traverse((child) => {
        if (child.isMesh) {
          child.castShadow = true;
          child.receiveShadow = true;
        }
      });

      // 组装场景：Model -> Group -> Scene
      butterflyGroup.add(butterflyModel);
      scene.add(butterflyGroup);

      // --- 动画剪辑处理 ---
      // 如果模型内置了动画（如：翅膀扇动），则启动 AnimationMixer 进行播放
      if (gltf.animations && gltf.animations.length > 0) {
        mixer = new THREE.AnimationMixer(butterflyModel);
        gltf.animations.forEach((clip) => {
          mixer.clipAction(clip).play();
        });
      }

      loading.value = false;
    },
    (xhr) => {
      progress.value = Math.floor((xhr.loaded / xhr.total) * 100);
    },
    (error) => {
      console.error('模型加载失败:', error);
      loading.value = false;
    }
  );

  // 6. 事件监听
  window.addEventListener('resize', onWindowResize);
  container.value.addEventListener('mousemove', onMouseMove);
};

/**
 * 鼠标移动回调：核心交互逻辑
 * 将屏幕上的 2D 坐标转换为 3D 世界中的目标飞行位置
 */
const onMouseMove = (event) => {
  // 1. 将鼠标点击的屏幕像素坐标转换为归一化设备坐标 (NDC)
  // 范围：x 从左到右 (-1 到 1)，y 从下到上 (-1 到 1)
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  // 2. 创建一个在投影平面上的向量
  const vector = new THREE.Vector3(mouse.x, mouse.y, 0.5);
  // 使用相机投影矩阵的逆矩阵，将屏幕点反投影回 3D 空间
  vector.unproject(camera);

  // 3. 计算从相机位置指向该反投影点的射线方向向量
  const dir = vector.sub(camera.position).normalize();

  // 4. 【核心投影逻辑】：
  // 我们假设蝴蝶是在一个虚拟的垂直平面（Z=0附近）飞行。
  // 通过相似三角形或射线方程计算，确定射线与该平面的交点。
  // distance 计算公式：射线在 Z 轴方向的分量与相机 Z 坐标的关系。
  const distance = -camera.position.z / dir.z;
  // 5. 更新目标点：将目标点设置在射线路径上，乘以 0.8 是为了让蝴蝶稍微靠近相机一点，不至于贴死在平面上
  targetPosition.copy(camera.position).add(dir.multiplyScalar(distance * 0.8));
};

/**
 * 渲染主循环：处理平滑动画和 AI 行为
 */
const animate = () => {
  animationId = requestAnimationFrame(animate);

  const delta = clock.getDelta();
  // 更新 GLTF 模型内置的骨骼动画（如翅膀扇动频率）
  if (mixer) mixer.update(delta);

  // --- 蝴蝶飞行 AI 交互逻辑 ---
  if (butterflyGroup && butterflyModel) {
    // 计算当前位置与目标点之间的实时距离
    const dist = butterflyGroup.position.distanceTo(targetPosition);

    // 【逻辑 1：平滑位移 (Lerp)】
    // 使用线性插值让蝴蝶平滑地靠近目标点。
    // followConfig.moveSpeed 决定了“紧随”程度：数值越小，延迟感和惯性越强。
    butterflyGroup.position.lerp(targetPosition, followConfig.moveSpeed);

    // 【逻辑 2：平滑转向 (Slerp)】
    // 我们不希望蝴蝶瞬间转身，而是平滑地转弯。
    const lookTarget = targetPosition.clone();
    if (dist > 0.2) { // 只有在距离足够远时才更新转向，避免到达终点时的微小抖动
      const tempMatrix = new THREE.Matrix4();
      // 使用 lookAt 计算出“面向目标”的旋转矩阵
      tempMatrix.lookAt(butterflyGroup.position, lookTarget, butterflyGroup.up);
      // 将矩阵转换为四元数（Quaternion），以便进行球面线性插值(Slerp)
      const targetQuaternion = new THREE.Quaternion().setFromRotationMatrix(tempMatrix);
      // 平滑旋转父容器 butterflyGroup
      butterflyGroup.quaternion.slerp(targetQuaternion, followConfig.turnSpeed);
    }

    // 【逻辑 3：动态仰角 (Pitch) 实现“仰望”效果】
    // 这是一个特殊的视觉效果：当蝴蝶飞近光标时，它会优雅地抬起头。
    let targetTiltX = 0;
    if (dist < followConfig.tiltThreshold) {
      // 距离越近，tiltFactor 越接近 1.0
      const tiltFactor = 1.0 - (dist / followConfig.tiltThreshold);
      // 目标仰角设定为 -45 度 (Math.PI / 4)。
      // 注意：在 Three.js 中，绕 X 轴负旋转通常代表抬头。
      targetTiltX = -(Math.PI / 4) * tiltFactor;
    }

    // 使用 lerp 对 butterflyModel（子对象）的局部 X 轴旋转进行插值。
    // 这样既能抬头，又不影响父容器 butterflyGroup 的水平转向。
    butterflyModel.rotation.x = THREE.MathUtils.lerp(
      butterflyModel.rotation.x,
      targetTiltX,
      followConfig.tiltSpeed
    );

    // 【逻辑 4：极近距离姿态修正】
    // 当蝴蝶几乎重合在光标位置时（dist < 0.3），lookAt 会因为方向向量过短而失效。
    // 此时我们通过 slerp 维持当前的四元数状态，防止蝴蝶在终点处因精度问题发生剧烈闪烁或乱转。
    if (dist < 0.3) {
      butterflyGroup.quaternion.slerp(butterflyGroup.quaternion, 0.1);
    }

    // 【逻辑 5：仿生随机晃动】
    // 叠加正弦和余弦函数，模拟蝴蝶在空中受微弱气流影响而产生的自然漂浮感。
    const time = Date.now() * 0.002;
    butterflyGroup.position.y += Math.sin(time) * 0.005;
    butterflyGroup.position.x += Math.cos(time * 0.5) * 0.005;
  }

  // 执行渲染
  renderer.render(scene, camera);
};

/**
 * 窗口尺寸变更处理：确保渲染画布始终充满屏幕
 */
const onWindowResize = () => {
  // 更新相机的宽高比
  camera.aspect = window.innerWidth / window.innerHeight;
  // 更新投影矩阵（必须调用）
  camera.updateProjectionMatrix();
  // 调整渲染器输出画布的大小
  renderer.setSize(window.innerWidth, window.innerHeight);
};

onMounted(() => {
  init();
  animate();
});

onUnmounted(() => {
  // 资源清理，防止内存泄漏
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onWindowResize);
  if (container.value) {
    container.value.removeEventListener('mousemove', onMouseMove);
  }
  if (renderer) {
    renderer.dispose();
    renderer.forceContextLoss();
  }
});
</script>

<style scoped>
.butterfly-container {
  width: 100vw;
  height: 100vh;
  background-color: #111122;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: #d4af37;
  z-index: 100;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 5px solid #333;
  border-top: 5px solid #d4af37;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 20px;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(360deg);
  }
}
</style>
