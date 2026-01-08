<template>
    <div ref="container" class="gui-test-container">
        <!-- 集成之前创建的性能监视器 -->
        <StatsMonitor ref="statsRef" :mode="0" :position="{ top: '10px', left: '10px' }" />

        <div class="info-overlay">
            <h3>GUI 交互测试</h3>
            <p>使用右上角的面板实时调整 3D 场景参数</p>
        </div>
    </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GUI } from 'three/examples/jsm/libs/lil-gui.module.min.js';
import StatsMonitor from './StatsMonitor.vue';

// --- 变量声明 ---
const container = ref(null);
const statsRef = ref(null);

let scene, camera, renderer, controls, animationId;
let cube, ambientLight, directionalLight, gridHelper;
let gui;

// GUI 控制参数对象
const params = {
    geometryType: 'Box',
    cubeColor: 0x00ff00,
    cubeScale: 1,
    cubePosX: 0,
    cubePosY: 0.5,
    cubePosZ: 0,
    wireframe: false,
    autoRotate: true,
    rotationSpeed: 0.01,
    showGrid: true,
    lightIntensity: 1.5,
    fogColor: 0x111122,
    reset: () => {
        // 1. 重置参数对象的值
        params.geometryType = 'Box';
        params.cubePosX = 0;
        params.cubePosY = 0.5;
        params.cubePosZ = 0;
        params.cubeScale = 1;
        params.autoRotate = true;
        params.rotationSpeed = 0.01;
        params.cubeColor = 0x00ff00;
        params.wireframe = false;
        params.showGrid = true;
        params.lightIntensity = 1.5;
        params.fogColor = 0x111122;

        // 2. 手动同步到 Three.js 场景对象
        updateGeometry('Box');
        if (cube) {
            cube.position.set(params.cubePosX, params.cubePosY, params.cubePosZ);
            cube.scale.setScalar(params.cubeScale);
            cube.material.color.set(params.cubeColor);
            cube.material.wireframe = params.wireframe;
        }
        if (gridHelper) {
            gridHelper.visible = params.showGrid;
        }
        if (directionalLight) {
            directionalLight.intensity = params.lightIntensity;
        }
        if (scene) {
            scene.background.set(params.fogColor);
            scene.fog.color.set(params.fogColor);
        }

        // 3. 刷新 GUI 界面显示
        refreshGUI();
    }
};

/**
 * 更新几何体类型 (下拉菜单演示)
 */
const updateGeometry = (type) => {
    if (!cube) return;

    // 销毁旧几何体释放内存
    cube.geometry.dispose();

    switch (type) {
        case 'Box':
            cube.geometry = new THREE.BoxGeometry(1, 1, 1);
            break;
        case 'Sphere':
            cube.geometry = new THREE.SphereGeometry(0.7, 32, 32);
            break;
        case 'Torus':
            cube.geometry = new THREE.TorusGeometry(0.5, 0.2, 16, 100);
            break;
    }
};

/**
 * 刷新 GUI 面板显示（当外部重置参数时调用）
 */
const refreshGUI = () => {
    if (gui) {
        gui.controllers.forEach(c => c.updateDisplay());
    }
};

/**
 * 初始化场景
 */
const init = () => {
    // 1. 场景与相机
    scene = new THREE.Scene();
    scene.background = new THREE.Color(params.fogColor);
    scene.fog = new THREE.Fog(params.fogColor, 1, 20);

    camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(3, 3, 5);

    // 2. 渲染器
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.shadowMap.enabled = true;
    container.value.appendChild(renderer.domElement);

    // 3. 控制器
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    // 4. 物体：地板与网格辅助线
    const floor = new THREE.Mesh(
        new THREE.PlaneGeometry(10, 10),
        new THREE.MeshStandardMaterial({ color: 0x444444 })
    );
    floor.rotation.x = -Math.PI / 2;
    floor.receiveShadow = true;
    scene.add(floor);

    gridHelper = new THREE.GridHelper(10, 10, 0x888888, 0x444444);
    gridHelper.visible = params.showGrid;
    scene.add(gridHelper);

    // 5. 物体：测试立方体
    const geometry = new THREE.BoxGeometry(1, 1, 1);
    const material = new THREE.MeshStandardMaterial({ color: params.cubeColor });
    cube = new THREE.Mesh(geometry, material);
    cube.position.y = 0.5;
    cube.castShadow = true;
    scene.add(cube);

    // 6. 灯光
    ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
    scene.add(ambientLight);

    directionalLight = new THREE.DirectionalLight(0xffffff, params.lightIntensity);
    directionalLight.position.set(5, 5, 5);
    directionalLight.castShadow = true;
    scene.add(directionalLight);

    // 7. 初始化 GUI
    initGUI();

    window.addEventListener('resize', onWindowResize);
};

/**
 * 初始化 GUI 控制面板
 */
const initGUI = () => {
    gui = new GUI({ title: '场景控制台' });

    // 立方体控制组
    const cubeFolder = gui.addFolder('物体设置');

    // 下拉菜单示例
    cubeFolder.add(params, 'geometryType', ['Box', 'Sphere', 'Torus'])
        .name('几何体类型')
        .onChange(val => updateGeometry(val));

    cubeFolder.add(params, 'cubePosX', -3, 3).step(0.1).name('位置 X').onChange(val => cube.position.x = val);
    cubeFolder.add(params, 'cubePosY', 0, 3).step(0.1).name('位置 Y').onChange(val => cube.position.y = val);
    cubeFolder.add(params, 'cubePosZ', -3, 3).step(0.1).name('位置 Z').onChange(val => cube.position.z = val);

    cubeFolder.add(params, 'cubeScale', 0.1, 3).step(0.1).name('缩放比例').onChange(val => {
        cube.scale.setScalar(val);
        // 根据几何体调整位置可能需要更复杂的逻辑，这里简化处理
    });

    cubeFolder.addColor(params, 'cubeColor').name('材质颜色').onChange(val => {
        cube.material.color.set(val);
    });

    cubeFolder.add(params, 'wireframe').name('显示线框').onChange(val => {
        cube.material.wireframe = val;
    });

    // 动画与环境组
    const envFolder = gui.addFolder('环境与动画');

    // 复选框示例
    envFolder.add(params, 'autoRotate').name('自动旋转');
    envFolder.add(params, 'showGrid').name('显示网格').onChange(val => {
        gridHelper.visible = val;
    });

    envFolder.add(params, 'rotationSpeed', 0, 0.1).step(0.1).name('旋转速度');
    envFolder.add(params, 'lightIntensity', 0, 5).step(0.1).name('光照强度').onChange(val => {
        directionalLight.intensity = val;
    });

    envFolder.addColor(params, 'fogColor').name('背景/雾颜色').onChange(val => {
        scene.background.set(val);
        scene.fog.color.set(val);
    });

    // 动作按钮
    gui.add(params, 'reset').name('重置所有参数');

    // 设置面板初始位置（可选）
    gui.domElement.style.top = '10px';
    gui.domElement.style.right = '10px';
};

const animate = () => {
    animationId = requestAnimationFrame(animate);

    // 更新性能监视器
    if (statsRef.value) statsRef.value.update();

    // 物体自转 (根据 checkbox 状态)
    if (cube && params.autoRotate) {
        cube.rotation.y += params.rotationSpeed;
        cube.rotation.x += params.rotationSpeed * 0.5;
    }

    controls.update();
    renderer.render(scene, camera);
};

const onWindowResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
};

onMounted(() => {
    init();
    animate();
});

onUnmounted(() => {
    cancelAnimationFrame(animationId);
    window.removeEventListener('resize', onWindowResize);

    // 销毁 GUI
    if (gui) gui.destroy();

    if (renderer) {
        renderer.dispose();
        if (renderer.domElement && renderer.domElement.parentNode) {
            renderer.domElement.parentNode.removeChild(renderer.domElement);
        }
    }
});
</script>

<style scoped>
.gui-test-container {
    width: 100vw;
    height: 100vh;
    background-color: #000;
    position: relative;
}

.info-overlay {
    position: absolute;
    bottom: 100px;
    left: 20px;
    color: white;
    pointer-events: none;
    background: rgba(0, 0, 0, 0.5);
    padding: 15px;
    border-radius: 8px;
    border-left: 4px solid #d4af37;
}

.info-overlay h3 {
    margin: 0 0 5px 0;
    color: #d4af37;
}

.info-overlay p {
    margin: 0;
    font-size: 14px;
    opacity: 0.8;
}

/* 深度覆盖 lil-gui 的层级，确保在最顶层 */
:deep(.lil-gui) {
    --width: 280px;
    z-index: 1001;
}
</style>
