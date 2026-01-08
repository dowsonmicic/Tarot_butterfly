<template>
  <div class="text-demo-container">
    <!-- Three.js 渲染容器 -->
    <div ref="container" class="canvas-container"></div>

    <!-- UI 信息面板 -->
    <div class="info-panel">
      <h1>Three.js 文字渲染演示</h1>
      <p>展示了官网提到的三种主要方式：</p>
      <ul>
        <li><strong>方式 1:</strong> TextGeometry (真正的 3D 几何体)</li>
        <li><strong>方式 2:</strong> CanvasTexture (2D 贴图，灵活性高)</li>
        <li><strong>方式 3:</strong> CSS2D (HTML 标签同步，适合 UI)</li>
      </ul>
      <button @click="$emit('back')">返回主场景</button>
    </div>

    <!-- 方式 3 的 HTML 元素示例 -->
    <div id="html-label" class="html-label">
      我是 HTML 标签 (跟随 3D 物体)
    </div>
  </div>
</template>

<script setup>
/**
 * -------------------------------------------------------------------------
 * 📝 Text.vue - Three.js 文字渲染教学组件
 * -------------------------------------------------------------------------
 * 根据 Three.js 官方手册 (https://threejs.org/manual/#en/creating-text) 编写。
 * 本组件旨在展示如何在 3D 场景中有效地处理文本。
 */
import { ref, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { FontLoader } from 'three/addons/loaders/FontLoader.js';
import { TextGeometry } from 'three/addons/geometries/TextGeometry.js';

const container = ref(null);
let scene, camera, renderer, controls, animationId;

// --- 方式 1: TextGeometry (3D 字体) ---
/**
 * 【深度解析：方式 1 - TextGeometry】
 * 
 * 原理：将字体文件（.json）中的矢量轮廓提取出来，通过“挤出”技术生成 3D 几何体。
 * 
 * 优点：
 * 1. 真正的 3D 效果：有厚度，有斜角，能反射光影，具有真实的体积感。
 * 2. 变换自由：可以像操作普通 3D 方块一样对它进行缩放、旋转。
 * 
 * 缺点：
 * 1. 性能开销：每个字母都会生成大量三角形面片，文字越多，渲染负担越重。
 * 2. 字体限制：必须先将字体（.ttf/otf）转换成 Three.js 特有的 .json 格式。
 */
const initTextGeometry = (font) => {
  // 创建 3D 文字形状
  const textGeo = new TextGeometry('Three.js 3D', {
    font: font,         // 字体对象
    size: 0.5,          // 文字大小（高度）
    depth: 0.2,         // 挤出厚度（Z轴方向）
    curveSegments: 12,  // 曲线的圆滑度（数值越高越圆滑，但面数越多）
    bevelEnabled: true, // 是否启用斜角（让文字边缘看起来更有倒角感）
    bevelThickness: 0.03, // 斜角的深度
    bevelSize: 0.02,    // 斜角离开轮廓的距离
    bevelOffset: 0,
    bevelSegments: 5    // 斜角的圆滑度
  });

  // 【知识点：居中对齐计算】
  // 默认情况下，文字的坐标原点在左下角。
  // 我们需要计算文字的包围盒（Bounding Box），将其中心点平移到原点。
  textGeo.computeBoundingBox();
  const centerOffset = -0.5 * (textGeo.boundingBox.max.x - textGeo.boundingBox.min.x);

  // MeshPhongMaterial 是一种受光照影响的材质，适合表现金属或塑料感
  const material = new THREE.MeshPhongMaterial({ color: 0x00ff88, specular: 0xffffff });
  const mesh = new THREE.Mesh(textGeo, material);

  mesh.position.x = centerOffset; // 应用水平居中偏移
  mesh.position.y = 1;            // 放在场景上方
  scene.add(mesh);
};

// --- 方式 2: Canvas 作为纹理 (2D Texture) ---
/**
 * 【深度解析：方式 2 - CanvasTexture】
 * 
 * 原理：在内存中创建一个隐藏的 HTML5 Canvas，用原生的 Canvas 2D API 绘图，
 *      然后将这张“图片”作为纹理贴在一个 3D 平面（PlaneGeometry）上。
 * 
 * 优点：
 * 1. 极其灵活：可以使用所有 Canvas 绘图技巧（渐变、阴影、多行文字、表情包）。
 * 2. 性能优异：本质上只是一张图片，渲染开销极低。
 * 3. 兼容性好：支持浏览器自带的所有字体，无需加载额外的 .json 字体包。
 * 
 * 缺点：
 * 1. 视角受限：因为它只是贴在一张纸上的图，从侧面看会“消失”。
 * 2. 分辨率问题：Canvas 尺寸固定后，相机拉近看会有模糊或锯齿。
 */
const initCanvasText = () => {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');

  // 设置 Canvas 尺寸（分辨率）
  canvas.width = 512;
  canvas.height = 128;

  // 1. 绘制背景（半透明黑色）
  ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // 2. 绘制文字样式
  ctx.font = 'bold 60px Arial';
  ctx.fillStyle = '#ffdd00'; // 亮黄色
  ctx.textAlign = 'center';  // 水平居中
  ctx.textBaseline = 'middle'; // 垂直居中

  // 3. 在 Canvas 中心写入文字
  ctx.fillText('Canvas 贴图文字', 256, 64);

  // 将 Canvas 转换为 Three.js 纹理
  const texture = new THREE.CanvasTexture(canvas);

  // 创建一个平面网格来显示这张“贴图”
  // MeshBasicMaterial 不受光照影响，确保文字清晰可见
  const material = new THREE.MeshBasicMaterial({
    map: texture,
    transparent: true // 开启透明，允许背景穿透
  });

  const geometry = new THREE.PlaneGeometry(4, 1); // 4x1 的矩形平面
  const mesh = new THREE.Mesh(geometry, material);

  mesh.position.y = -1; // 放在场景下方
  scene.add(mesh);
};

// --- 初始化场景 ---
/**
 * 标准的 Three.js 初始化流程
 */
const init = () => {
  // 1. 创建场景并设置深蓝色背景
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111122);

  // 2. 设置透视相机
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(0, 0, 5); // 相机后退 5 个单位，看清全局

  // 3. 创建 WebGL 渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  container.value.appendChild(renderer.domElement); // 挂载到 Vue 的 ref 容器

  // 4. 添加轨道控制器（鼠标旋转、缩放）
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true; // 开启阻尼感，让旋转更平滑

  // 5. 【重要】添加光照
  // 3D 文字（方式 1）使用了 MeshPhongMaterial，必须有光才能看见。
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6); // 环境光：照亮暗部
  scene.add(ambientLight);
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);    // 平行光：产生高光和体积感
  dirLight.position.set(5, 5, 5);
  scene.add(dirLight);

  // 6. 加载字体并初始化 3D 文字
  // FontLoader 用于解析 Three.js 专用的 JSON 字体文件
  const loader = new FontLoader();
  loader.load('https://threejs.org/examples/fonts/helvetiker_bold.typeface.json', (font) => {
    initTextGeometry(font);
  });

  // 7. 初始化 Canvas 文字
  initCanvasText();

  // 8. 添加网格辅助线（作为空间参考，让我们知道哪里是地面）
  const axesHelper = new THREE.AxesHelper(5);
  scene.add(axesHelper);
  const grid = new THREE.GridHelper(10, 10, 0x444444, 0x222222);
  grid.position.y = -2;
  scene.add(grid);

  animate();
};

/**
 * 每一帧的渲染逻辑
 */
const animate = () => {
  animationId = requestAnimationFrame(animate);
  controls.update(); // 必须在每一帧调用，才能让阻尼（Damping）生效
  renderer.render(scene, camera);
};

const onResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
};

onMounted(() => {
  init();
  window.addEventListener('resize', onResize);
});

onUnmounted(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onResize);

  if (renderer) {
    renderer.dispose();
    if (renderer.domElement && renderer.domElement.parentNode) {
      renderer.domElement.parentNode.removeChild(renderer.domElement);
    }
  }
});
</script>

<style scoped>
.text-demo-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
}

.canvas-container {
  width: 100%;
  height: 100%;
}

.info-panel {
  position: absolute;
  top: 20px;
  left: 20px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 20px;
  border-radius: 12px;
  border: 1px solid #444;
  backdrop-filter: blur(10px);
  max-width: 300px;
  pointer-events: auto;
  z-index: 100;
}

h1 {
  font-size: 1.2rem;
  margin-bottom: 10px;
  color: #00ff88;
}

p {
  font-size: 0.9rem;
  color: #ccc;
}

ul {
  padding-left: 20px;
  margin: 10px 0;
}

li {
  font-size: 0.85rem;
  margin-bottom: 5px;
}

button {
  margin-top: 10px;
  padding: 8px 16px;
  background: #00ff88;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

button:hover {
  background: #00cc6e;
}

.html-label {
  position: absolute;
  background: white;
  color: black;
  padding: 5px 10px;
  border-radius: 4px;
  font-size: 12px;
  pointer-events: none;
  /* 
     注意：在真实项目中，CSS2D 需要特殊的 CSS2DRenderer。
     这里仅作为概念演示，固定在屏幕位置或通过计算映射。
  */
  bottom: 50px;
  right: 50px;
  box-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}
</style>
