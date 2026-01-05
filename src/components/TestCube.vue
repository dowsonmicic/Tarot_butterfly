<template>
  <!-- 
    ref="container" 是 Vue 的模板引用。
    在脚本中通过 const container = ref(null) 来对接，
    使得我们可以直接操作这个真实的 <div> 元素，
    它是 Three.js 渲染器渲染出的 <canvas> 画布的“家”。
  -->
  <div ref="container" class="test-cube-container"></div>
</template>

<script setup>
/**
 * 导入必要的 Vue 和 Three.js 模块
 */
import { onMounted, onUnmounted, ref } from 'vue';
import * as THREE from 'three'; // 引入整个 Three.js 核心库
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'; // 引入轨道控制器，允许用鼠标旋转相机

// 声明变量，用于存储 Three.js 的核心组件
const container = ref(null); // 对应模板中的 div 容器
let scene;        // 场景：所有 3D 物体的“世界”
let camera;       // 相机：我们的“眼睛”，决定看到什么
let renderer;     // 渲染器：负责把 3D 数据画成 2D 像素并显示在屏幕上
let cube;         // 我们要画的立方体
let controls;     // 鼠标控制器
let animationId;  // 记录动画请求的 ID，方便在组件销毁时停止动画

/**
 * 初始化 Three.js 环境
 */
const init = () => {
  // 1. 创建场景 (Scene)
  // 这是所有物体的容器，就像一个空的宇宙。
  scene = new THREE.Scene();

  // 2. 创建相机 (Camera)
  // PerspectiveCamera (透视相机) 模拟人眼的视觉效果（近大远小）。
  // 参数解释：
  // 75: 视野角度 (FOV)，单位是度。
  // window.innerWidth / window.innerHeight: 画布的长宽比。
  // 0.1: 近剪切面，比这个距离近的物体不会被渲染。
  // 1000: 远剪切面，比这个距离远的物体不会被渲染。
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 5; // 将相机向后移动 5 个单位，否则相机会在立方体中心

  // 3. 创建渲染器 (Renderer)
  // antialias: true 开启抗锯齿，让物体边缘看起来更平滑，不刺眼。
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight); // 设置渲染区域的大小为全屏
  // 将渲染器自动生成的 <canvas> 元素放入我们在模板里定义的 div 容器中
  container.value.appendChild(renderer.domElement);

  // 4. 创建控制器 (OrbitControls)
  // 允许你用鼠标左键旋转、中键缩放、右键平移。
  controls = new OrbitControls(camera, renderer.domElement);

  // 5. 创建立方体 (Mesh)
  // 一个 Mesh (网格) 由几何体 (Geometry) 和材质 (Material) 组成。
  // BoxGeometry(1, 1, 1) 创建一个长宽高都为 1 的立方体形状数据。
  const geometry = new THREE.BoxGeometry(1, 1, 1);
  // MeshBasicMaterial 是最基础的材质，不受光照影响。
  // color: 0x00ff00 设置为纯绿色。
  // wireframe: true 开启线框模式，只显示骨架，不填充表面。
  const material = new THREE.MeshBasicMaterial({ color: 0x00ff00, wireframe: true });

  // 把形状和材质组合在一起，生成真正的 3D 物体
  cube = new THREE.Mesh(geometry, material);
  scene.add(cube); // 将立方体放入场景中，否则它是看不见的

  /**
   * 动画循环函数 (Animation Loop)
   * 这个函数会以极快的速度（通常每秒 60 次）被反复调用
   */
  const animate = () => {
    // 告诉浏览器：在下次重绘之前，再次调用这个 animate 函数
    // 这样就形成了一个无限循环，从而产生平滑的动画效果
    animationId = requestAnimationFrame(animate);

    /**
     * 【详细解释立方体旋转逻辑】
     * 
     * cube.rotation.x 代表立方体绕着它的 X 轴（横向轴）旋转的角度。
     * cube.rotation.y 代表立方体绕着它的 Y 轴（纵向轴）旋转的角度。
     * 
     * 这里的数值（如 0.01）单位是【弧度】，而不是【角度】。
     * 知识点：一个圆周（360度）等于 2π 弧度（约 6.28）。
     * 
     * 它是如何转起来的？
     * 1. 累加逻辑：使用 += 符号。每一帧我们都在当前角度的基础上增加 0.01 弧度。
     * 2. 持续运动：由于 animate 每秒执行约 60 次，
     *    所以每秒钟增加的角度就是 0.01 * 60 = 0.6 弧度（约 34.4 度）。
     * 3. 视觉效果：
     *    - 增加 x：立方体就像在做“翻筋斗”，绕着水平轴转动。
     *    - 增加 y：立方体就像在“原地转圈”，绕着垂直轴转动。
     * 两者结合，就形成了这种斜向的、看起来比较随机且动感的旋转效果。
     */
    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;

    // 更新轨道控制器（如果你手动用鼠标拖拽了，这里会同步相机位置）
    controls.update();

    // 核心步骤：让渲染器按照当前的相机视角，把场景拍成一张照片画在屏幕上
    renderer.render(scene, camera);
  };

  // 启动第一次动画
  animate();

  // 7. 监听窗口大小变化
  // 当用户拉大或缩小浏览器窗口时，我们需要同步更新画面
  window.addEventListener('resize', onWindowResize);
};

/**
 * 窗口缩放适配函数
 */
const onWindowResize = () => {
  // 1. 更新相机的长宽比，防止画面被压扁或拉长
  camera.aspect = window.innerWidth / window.innerHeight;
  // 2. 更新相机的投影矩阵（必须调用这个方法，相机才会应用新的长宽比）
  camera.updateProjectionMatrix();
  // 3. 更新渲染器的尺寸，让画布始终填满窗口
  renderer.setSize(window.innerWidth, window.innerHeight);
};

/**
 * Vue 生命周期：组件挂载完成
 * 此时 template 里的 HTML 已经渲染好了，我们可以安全地操作 DOM。
 */
onMounted(() => {
  init();
});

/**
 * Vue 生命周期：组件销毁之前
 * 当你关闭页面或切换到其他组件时触发。
 * 这是为了“善后”，清理内存，防止程序在后台继续跑，导致浏览器变卡。
 */
onUnmounted(() => {
  // 1. 停止动画请求，不再调用 animate 函数
  cancelAnimationFrame(animationId);

  // 2. 移除窗口缩放的监听器
  window.removeEventListener('resize', onWindowResize);

  // 3. 深度清理 Three.js 资源
  if (renderer) {
    renderer.dispose(); // 释放渲染器占用的 GPU 资源
    // 安全地将画布从网页中移除
    if (renderer.domElement && renderer.domElement.parentNode) {
      renderer.domElement.parentNode.removeChild(renderer.domElement);
    }
  }
});
</script>

<style scoped>
.test-cube-container {
  /* 强制容器占满整个浏览器视口 */
  width: 100%;
  height: 100vh;

  /* 固定定位在最顶层，不随滚动条滚动 */
  position: fixed;
  top: 0;
  left: 0;

  /* z-index 确保它盖在所有内容上面，方便我们看效果 */
  z-index: 999;

  /* 背景设为纯黑，衬托绿色的立方体 */
  background: #000;
}
</style>
