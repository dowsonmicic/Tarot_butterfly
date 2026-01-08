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
 * 导入必要的 Vue 和 Three.js 核心库
 */
import { onMounted, onUnmounted, ref } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'; // GLTF 模型加载器

// 使用 Vite 的 ?url 特性获取 3D 模型的正确引用路径
import modelUrl from '../../resource/borboleta_azul_-_butterfly/scene.gltf?url';

// --- 响应式状态与变量声明 ---
const container = ref(null); // 承载 Three.js 画布的 DOM 容器
const loading = ref(true);     // 控制加载遮罩层的显示
const progress = ref(0);      // 模型加载百分比进度

// Three.js 核心对象（非响应式，避免性能损耗）
let scene;          // 场景容器
let camera;         // 透视相机
let renderer;       // WebGL 渲染器
let animationId;    // 动画循环 ID
let mixer;          // 模型动画混合器（处理翅膀煽动等动画）
let butterflyGroup; // 蝴蝶的父级组对象，用于控制整体飞行位移与朝向
let butterflyModel; // 实际的蝴蝶网格模型，用于内部姿态微调（如抬头动作）
let clock = new THREE.Clock();           // 时间跟踪器，用于保持动画步频一致
let mouse = new THREE.Vector2();         // 存储归一化的鼠标坐标
let targetPosition = new THREE.Vector3(); // 蝴蝶追逐的目标 3D 坐标

/**
 * 飞行与交互参数配置
 * 通过调整这些数值可以改变蝴蝶的飞行手感
 */
const followConfig = {
  moveSpeed: 0.05,    // 移动平滑系数：值越大，跟随越紧，反应越快
  turnSpeed: 0.05,    // 转向平滑系数：值越大，转弯越灵敏
  tiltSpeed: 0.05,    // 抬头动作速度：值越大，仰望动作越迅猛
  tiltThreshold: 3.0  // 仰望触发距离：蝴蝶距离光标小于此值时开始抬头
};

/**
 * 初始化 Three.js 场景环境
 */
const init = () => {
  // 1. 场景设置
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111122); // 深蓝色夜空背景
  // 添加雾化效果：模拟远近的空间感
  scene.fog = new THREE.Fog(0x111122, 5, 15);

  // 2. 相机设置
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(0, 1, 5); // 将相机稍微抬高并后移

  // 3. 渲染器设置
  renderer = new THREE.WebGLRenderer({ antialias: true }); // 开启抗锯齿
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio); // 适配高分辨率屏幕
  renderer.shadowMap.enabled = true; // 开启阴影渲染
  renderer.outputColorSpace = THREE.SRGBColorSpace; // 设置标准色彩输出
  container.value.appendChild(renderer.domElement); // 将画布加入 DOM

  // 4. 灯光系统
  // 环境光：为场景提供基础的全局照明
  const ambientLight = new THREE.AmbientLight(0xffffff, 1.5);
  scene.add(ambientLight);

  // 平行光：模拟月光或远方光源，用于产生阴影和立体感
  const directionalLight = new THREE.DirectionalLight(0xffffff, 2);
  directionalLight.position.set(5, 5, 5);
  directionalLight.castShadow = true; // 开启光源产生阴影的能力
  scene.add(directionalLight);

  // 5. 模型加载逻辑
  const loader = new GLTFLoader();

  loader.load(
    modelUrl,
    (gltf) => {
      butterflyModel = gltf.scene;

      // 创建一个逻辑容器（Group），将模型包裹在内
      // 这样做的好处是：Group 负责整体移动，内部 Model 负责姿态旋转，互不干扰
      butterflyGroup = new THREE.Group();

      // 计算模型包围盒，确保无论模型原始大小如何，都能自动适配比例
      const box = new THREE.Box3().setFromObject(butterflyModel);
      const center = box.getCenter(new THREE.Vector3());
      const size = box.getSize(new THREE.Vector3());

      const maxDim = Math.max(size.x, size.y, size.z);
      const scale = 0.8 / maxDim; // 统一缩放到约 0.8 个单位大小
      butterflyModel.scale.setScalar(scale);

      // 将模型几何中心对齐到组的原点
      butterflyModel.position.sub(center.multiplyScalar(scale));

      /**
       * 【模型朝向校正】
       * 不同模型在制作时坐标轴定义不同。
       * 我们的逻辑假设蝴蝶头部朝向 Z 轴正方向，如果不是，需要在此调整。
       */
      butterflyModel.rotation.y = -Math.PI / 2;

      // --- 5. 遍历模型：开启阴影与材质优化 ---
      /**
       * butterflyModel.traverse 是 Three.js 提供的深度优先遍历函数。
       * 它会从根节点开始，递归地访问模型中的每一个零件（子物体）。
       * @param child - 代表遍历到的当前物体。在 3D 模型中，它可能是一个 Group（组）、
       *                Mesh（网格/真正的几何体）或者 Bone（骨骼）。
       */
      butterflyModel.traverse((child) => {
        // 判断当前零件是否为网格物体（Mesh）
        // 只有 Mesh 才能投射阴影和接收阴影
        if (child.isMesh) {
          child.castShadow = true;    // 蝴蝶会往别的物体上投射阴影
          child.receiveShadow = true; // 蝴蝶身体也能接收光照产生的阴影，增加立体感
        }
      });

      butterflyGroup.add(butterflyModel);
      scene.add(butterflyGroup);

      // --- 6. 动画系统：驱动蝴蝶煽动翅膀 ---
      /**
       * gltf.animations 包含模型自带的所有动画剪辑数据（如：翅膀上下煽动的动作）。
       */
      if (gltf.animations && gltf.animations.length > 0) {
        /**
         * AnimationMixer 是 Three.js 的“动画播放器”。
         * 它负责管理特定模型（butterflyModel）上的所有动画播放、混合和时间缩放。
         */
        mixer = new THREE.AnimationMixer(butterflyModel);

        /**
         * 遍历每一个动画剪辑（AnimationClip）。
         * @param clip - 代表一个具体的动作序列（比如“翅膀煽动”或“触角晃动”）。
         *               它包含了关键帧数据，告诉程序在什么时间点物体应该是什么姿态。
         */
        gltf.animations.forEach((clip) => {
          /**
           * mixer.clipAction(clip) 将原始的动画数据（clip）包装成一个可操作的“动作控制器”（AnimationAction）。
           * .play() 则是启动这个动作，默认会循环播放。
           */
          mixer.clipAction(clip).play();
        });
      }

      loading.value = false; // 加载完成，隐藏 UI 遮罩
    },
    /**
     * 加载过程中的进度回调
     * @param xhr - 这是一个 ProgressEvent 对象（XMLHttpRequest 进度事件）。
     *              它包含了加载过程中的字节信息，用于计算百分比。
     */
    (xhr) => {
      // xhr.loaded: 已经下载的字节数
      // xhr.total: 文件总共的字节数
      if (xhr.total > 0) {
        progress.value = Math.floor((xhr.loaded / xhr.total) * 100);
      }
    },
    (error) => {
      console.error('模型加载失败:', error);
      loading.value = false;
    }
  );

  // 6. 注册交互事件
  window.addEventListener('resize', onWindowResize);
  container.value.addEventListener('mousemove', onMouseMove);
};

/**
 * 鼠标移动处理函数
 * 将 2D 屏幕坐标转换为 3D 空间中的目标落点
 */
const onMouseMove = (event) => {
  // 归一化设备坐标 (NDC)：范围为 [-1, 1]
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  // 将鼠标位置投影到相机前方的虚拟平面上
  const vector = new THREE.Vector3(mouse.x, mouse.y, 0.5);
  vector.unproject(camera);
  const dir = vector.sub(camera.position).normalize();

  // 计算目标点：在相机前方一定距离处追逐
  const distance = -camera.position.z / dir.z;
  targetPosition.copy(camera.position).add(dir.multiplyScalar(distance * 0.8));
};

/**
 * 窗口缩放适配
 */
const onWindowResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
};

/**
 * 核心动画循环
 */
const animate = () => {
  animationId = requestAnimationFrame(animate);

  const delta = clock.getDelta(); // 获取两帧之间的时间间隔
  if (mixer) mixer.update(delta); // 推进骨骼动画进度

  // --- 蝴蝶智能跟随算法 ---
  if (butterflyGroup && butterflyModel) {
    // 计算蝴蝶当前位置与鼠标目标点的距离
    const dist = butterflyGroup.position.distanceTo(targetPosition);

    // 1. 位置跟随：使用 lerp（线性插值）实现平滑移动
    butterflyGroup.position.lerp(targetPosition, followConfig.moveSpeed);

    // 2. 转向逻辑
    const lookTarget = targetPosition.clone();
    if (dist > 0.2) {
      // 当距离较远时，让蝴蝶父容器执行 lookAt 转向
      const tempMatrix = new THREE.Matrix4();
      tempMatrix.lookAt(butterflyGroup.position, lookTarget, butterflyGroup.up);
      const targetQuaternion = new THREE.Quaternion().setFromRotationMatrix(tempMatrix);
      // 使用 slerp（球面线性插值）让转向过程丝滑不僵硬
      butterflyGroup.quaternion.slerp(targetQuaternion, followConfig.turnSpeed);
    }

    // 3. 仰望姿态控制（特色逻辑）
    let targetTiltX = 0;
    if (dist < followConfig.tiltThreshold) {
      // 距离越近，仰望强度越高
      const tiltFactor = 1.0 - (dist / followConfig.tiltThreshold);
      // 目标仰角：最大 45 度
      targetTiltX = (Math.PI / 4) * tiltFactor;
    }

    // 平滑应用抬头动作
    butterflyModel.rotation.x = THREE.MathUtils.lerp(
      butterflyModel.rotation.x,
      targetTiltX,
      followConfig.tiltSpeed
    );

    // 4. 近距离稳定性优化
    if (dist < 0.3) {
      // 当极度靠近光标时，减弱旋转更新，防止产生剧烈的抖动
      butterflyGroup.quaternion.slerp(butterflyGroup.quaternion, 0.1);
    }

    // 5. 环境氛围：添加随机的小幅上下晃动，模拟昆虫飞行的悬停感
    const time = Date.now() * 0.002;
    butterflyGroup.position.y += Math.sin(time) * 0.005;
    butterflyGroup.position.x += Math.cos(time * 0.5) * 0.005;
  }

  // 执行渲染绘制
  renderer.render(scene, camera);
};

/**
 * 组件生命周期钩子
 */
onMounted(() => {
  init();
  animate();
});

onUnmounted(() => {
  // 必须进行的资源清理，防止内存泄漏
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onWindowResize);

  if (container.value) {
    container.value.removeEventListener('mousemove', onMouseMove);
  }

  if (renderer) {
    renderer.dispose(); // 销毁 WebGL 渲染上下文
    renderer.forceContextLoss();
    // 安全地从 DOM 中移除 canvas 画布
    if (renderer.domElement && renderer.domElement.parentNode) {
      renderer.domElement.parentNode.removeChild(renderer.domElement);
    }
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
