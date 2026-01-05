<template>
  <div ref="container" class="butterfly-container">
    <div class="loading-overlay" v-if="loading">
      <div class="spinner"></div>
      <p>正在加载蝴蝶模型... {{ progress }}%</p>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

// 使用 Vite 的 ?url 后缀来获取静态资源的正确路径
import modelUrl from '../../resource/borboleta_azul_-_butterfly/scene.gltf?url';

const container = ref(null);
const loading = ref(true);
const progress = ref(0);

let scene, camera, renderer, animationId, mixer;
let butterflyGroup; // 蝴蝶的父容器，用于位置和转向逻辑
let butterflyModel; // 实际的模型引用，用于初始朝向校正
let clock = new THREE.Clock();
let mouse = new THREE.Vector2();
let targetPosition = new THREE.Vector3();
let raycaster = new THREE.Raycaster();

// 飞行交互配置
const followConfig = {
  moveSpeed: 0.05,    // 移动速度：数值越大，蝴蝶跟得越紧
  turnSpeed: 0.05,     // 转向速度：数值越大，转弯越灵敏
  tiltSpeed: 0.05,    // 仰角速度：数值越大，抬头动作越快
  tiltThreshold: 3.0  // 触发抬头动作的距离阈值
};

/**
 * 初始化场景
 */
const init = () => {
  // 1. 场景与背景
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111122);
  // 添加雾化效果，增加深度感
  scene.fog = new THREE.Fog(0x111122, 5, 15);

  // 2. 相机
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(0, 1, 5);

  // 3. 渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  // 启用阴影渲染
  renderer.shadowMap.enabled = true;
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  container.value.appendChild(renderer.domElement);

  // 4. 光照
  // 环境光：基础亮度
  const ambientLight = new THREE.AmbientLight(0xffffff, 1.5);
  scene.add(ambientLight);

  // 平行光：模拟阳光，产生阴影
  const directionalLight = new THREE.DirectionalLight(0xffffff, 2);
  directionalLight.position.set(5, 5, 5);
  directionalLight.castShadow = true;
  scene.add(directionalLight);

  // 6. 加载模型
  const loader = new GLTFLoader();

  loader.load(
    modelUrl,
    (gltf) => {
      butterflyModel = gltf.scene;

      // 创建一个父容器
      butterflyGroup = new THREE.Group();

      // 自动居中并调整大小
      const box = new THREE.Box3().setFromObject(butterflyModel);
      const center = box.getCenter(new THREE.Vector3());
      const size = box.getSize(new THREE.Vector3());

      const maxDim = Math.max(size.x, size.y, size.z);
      const scale = 0.8 / maxDim;
      butterflyModel.scale.setScalar(scale);

      // 将模型移动到本地坐标原点
      butterflyModel.position.sub(center.multiplyScalar(scale));

      // 【朝向校正】：如果蝴蝶之前是屁股对着鼠标（-Z方向）
      // 我们需要再旋转 180 度，即从 Math.PI / 2 改为 -Math.PI / 2
      butterflyModel.rotation.y = -Math.PI / 2;

      // 遍历模型，开启阴影
      butterflyModel.traverse((child) => {
        if (child.isMesh) {
          child.castShadow = true;
          child.receiveShadow = true;
        }
      });

      // 将模型放入组中，再将组放入场景
      butterflyGroup.add(butterflyModel);
      scene.add(butterflyGroup);

      // 处理动画
      if (gltf.animations && gltf.animations.length > 0) {
        mixer = new THREE.AnimationMixer(butterflyModel);
        gltf.animations.forEach((clip) => {
          mixer.clipAction(clip).play();
        });
      }

      loading.value = false;
    },
    (xhr) => {
      // 加载进度
      progress.value = Math.floor((xhr.loaded / xhr.total) * 100);
    },
    (error) => {
      console.error('模型加载失败:', error);
      loading.value = false;
    }
  );

  // 7. 窗口调整监听
  window.addEventListener('resize', onWindowResize);
  // 8. 鼠标交互监听
  container.value.addEventListener('mousemove', onMouseMove);
};

const onMouseMove = (event) => {
  // 转换鼠标坐标到归一化设备坐标 (NDC)
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  // 将鼠标坐标映射到 3D 空间的一个平面上（z=2 的平面）
  const vector = new THREE.Vector3(mouse.x, mouse.y, 0.5);
  vector.unproject(camera);
  const dir = vector.sub(camera.position).normalize();
  const distance = -camera.position.z / dir.z; // 假设蝴蝶在 Z=0 附近飞行
  targetPosition.copy(camera.position).add(dir.multiplyScalar(distance * 0.8));
};

const onWindowResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
};

const animate = () => {
  animationId = requestAnimationFrame(animate);

  const delta = clock.getDelta();
  if (mixer) mixer.update(delta); // 更新动画混合器

  // 蝴蝶跟随逻辑
  if (butterflyGroup && butterflyModel) {
    const dist = butterflyGroup.position.distanceTo(targetPosition);

    // 1. 平滑移动位置 (Lerp)
    butterflyGroup.position.lerp(targetPosition, followConfig.moveSpeed);

    // 2. 平滑转向与仰角处理
    const lookTarget = targetPosition.clone();

    if (dist > 0.2) {
      // 基础水平转向
      const tempMatrix = new THREE.Matrix4();
      tempMatrix.lookAt(butterflyGroup.position, lookTarget, butterflyGroup.up);
      const targetQuaternion = new THREE.Quaternion().setFromRotationMatrix(tempMatrix);
      butterflyGroup.quaternion.slerp(targetQuaternion, followConfig.turnSpeed);
    }

    // 3. 实现“斜45度上仰”效果
    // 当距离小于 threshold 时，平滑切换到绝对的上仰姿态
    let targetTiltX = 0;

    if (dist < followConfig.tiltThreshold) {
      const tiltFactor = 1.0 - (dist / followConfig.tiltThreshold);
      // 目标仰角：45度 (Math.PI / 4)
      targetTiltX = (Math.PI / 4) * tiltFactor;

      // 【关键逻辑】：当距离极近时，我们希望蝴蝶“站起来”
      // 这里的 rotation.x 是相对于父容器的旋转
      // 随着 tiltFactor 增加，蝴蝶会从平飞逐渐转为仰望
    }

    // 平滑插值仰角
    butterflyModel.rotation.x = THREE.MathUtils.lerp(
      butterflyModel.rotation.x,
      targetTiltX,
      followConfig.tiltSpeed
    );

    // 【新增核心修复】：当蝴蝶几乎到达光标位置时，
    // 我们需要减弱其水平转向的限制，让它能保持住仰望的姿态，而不是被强制拉向水平看向光标
    if (dist < 0.3) {
      // 极近距离时，让蝴蝶的水平偏航（Yaw）也稍微平滑一点，不要剧烈晃动
      butterflyGroup.quaternion.slerp(butterflyGroup.quaternion, 0.1);
    }

    // 4. 增加一点随机的小晃动
    const time = Date.now() * 0.002;
    butterflyGroup.position.y += Math.sin(time) * 0.005;
    butterflyGroup.position.x += Math.cos(time * 0.5) * 0.005;
  }

  renderer.render(scene, camera);
};

onMounted(() => {
  init();
  animate();
});

onUnmounted(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onWindowResize);
  if (container.value) {
    container.value.removeEventListener('mousemove', onMouseMove);
  }
  if (renderer) {
    renderer.dispose();
    renderer.forceContextLoss();
    // 安全地将画布从网页中移除
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
