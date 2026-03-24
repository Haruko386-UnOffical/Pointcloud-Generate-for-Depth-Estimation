<template>
    <div class="scene-container">
        <canvas ref="canvasRef" class="webgl-canvas"></canvas>

        <div class="ui-panel">
            <div class="panel-header">
                <h1>3D POINT<br />CLOUD</h1>
                <div class="status-indicator">
                    <span class="dot" :class="{ active: hasDepth }"></span>
                    {{ hasDepth ? '3D MODE' : '2D MODE' }}
                </div>
            </div>

            <div class="divider"></div>

            <div class="controls-group">
                <div class="control-item">
                    <label class="label-text">TEXTURE (RGB)</label>
                    <input type="file" id="fileInput" accept="image/*" @change="handleFileUpload" />
                    <label for="fileInput" class="upload-btn main-btn">
                        <span class="icon">image</span>
                        <span v-if="loading">PROCESSING...</span>
                        <span v-else>UPLOAD IMAGE</span>
                    </label>
                </div>

                <div class="control-item">
                    <div class="label-row">
                        <label class="label-text">DEPTH MAP</label>
                        <span v-if="hasDepth" class="badge">ACTIVE</span>
                    </div>
                    <input type="file" id="depthInput" accept="image/*" @change="handleDepthUpload" />
                    <label for="depthInput" class="upload-btn sub-btn" :class="{ 'has-file': hasDepth }">
                        <span class="icon">layers</span>
                        <span>{{ hasDepth ? 'REPLACE DEPTH' : 'UPLOAD DEPTH' }}</span>
                    </label>

                    <div v-if="hasDepth" class="depth-tools">
                        <div class="tool-btn" @click.stop="toggleDepthInvert" :class="{ active: invertDepth }"
                            title="Invert Depth">
                            <span class="icon small">flip_camera_android</span>
                            <span class="tool-text">REV</span>
                        </div>
                        <div class="tool-btn remove" @click.stop="clearDepth" title="Remove Depth">
                            <span class="icon small">close</span>
                        </div>
                    </div>
                </div>
            </div>

            <p class="hint">LEFT CLICK: ROTATE | RIGHT CLICK: PAN | SCROLL: ZOOM</p>
        </div>
    </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { loadGifFrames } from '../utils/gifLoader';
import defaultImg from '@/assets/example_01.jpg';
import defaultDepthImg from '@/assets/example_01_depth.png';

const vertexShader = `
  uniform vec2 uTextureSize;
  uniform sampler2D uDepthTexture;
  uniform float uHasDepth;
  uniform float uDepthScale;
  uniform float uInvertDepth;
  uniform float uPointSize;

  varying vec2 vuv;

  void main() {
    vuv = uv;
    vec3 currentPos = position;
    float z = 0.0;

    if (uHasDepth > 0.5) {
        // 3D 模式：采样深度图
        float depthVal = texture2D(uDepthTexture, vuv).r;
        
        float finalDepth = 0.0;
        if (uInvertDepth > 0.5) {
            finalDepth = depthVal; 
        } else {
            finalDepth = 1.0 - depthVal;
        }

        z = finalDepth * uDepthScale - (uDepthScale * 0.5);
    } 

    vec3 newPosition = vec3(currentPos.x, currentPos.y, z);
    vec4 mvPosition = modelViewMatrix * vec4(newPosition, 1.0);
    gl_Position = projectionMatrix * mvPosition;
    gl_PointSize = uPointSize * (800.0 / -mvPosition.z);
  }
`;

const fragmentShader = `
  precision highp float;
  uniform sampler2D uTexture;
  varying vec2 vuv;
  
  void main() {
    vec4 color = texture2D(uTexture, vuv);
    if (color.a < 0.1) discard;
    gl_FragColor = color;
  }
`;

const canvasRef = ref(null);
const loading = ref(false);
const hasDepth = ref(false);
const invertDepth = ref(false);
const currentDepthTex = ref(null);

let renderer, scene, camera, controls, particlesMesh, animationId;
let gifState = { isGif: false, frames: [], currentFrameIndex: 0, lastFrameTime: 0, ctx: null, texture: null };

const state = {
    depthScale: 300,
    pointSize: 1.5
};

const toggleDepthInvert = () => {
    invertDepth.value = !invertDepth.value;
    if (particlesMesh) {
        particlesMesh.material.uniforms.uInvertDepth.value = invertDepth.value ? 1.0 : 0.0;
    }
};

const initScene = () => {
    const width = window.innerWidth;
    const height = window.innerHeight;
    renderer = new THREE.WebGLRenderer({ canvas: canvasRef.value, antialias: true, alpha: true });
    renderer.setSize(width, height);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

    scene = new THREE.Scene();
    scene.background = new THREE.Color(0xffffff);

    camera = new THREE.PerspectiveCamera(50, width / height, 0.1, 5000);
    camera.position.set(0, 0, 600);

    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.rotateSpeed = 0.5;
};

const createParticles = (width, height, texture) => {
    if (particlesMesh) {
        scene.remove(particlesMesh);
        particlesMesh.geometry.dispose();
        particlesMesh.material.dispose();
    }

    const geometry = new THREE.BufferGeometry();
    const positions = [];
    const uvs = [];

    const numParticles = width * height;
    const halfWidth = width / 2;
    const halfHeight = height / 2;

    for (let i = 0; i < numParticles; i++) {
        const x = (i % width);
        const y = Math.floor(i / width);

        positions.push(x - halfWidth, halfHeight - y, 0);
        uvs.push(x / width, 1.0 - y / height);
    }

    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    geometry.setAttribute('uv', new THREE.Float32BufferAttribute(uvs, 2));

    const material = new THREE.ShaderMaterial({
        uniforms: {
            uTexture: { value: texture },
            uDepthTexture: { value: currentDepthTex.value || new THREE.Texture() },
            uHasDepth: { value: hasDepth.value ? 1.0 : 0.0 },
            uDepthScale: { value: state.depthScale },
            uInvertDepth: { value: invertDepth.value ? 1.0 : 0.0 },
            uPointSize: { value: state.pointSize * window.devicePixelRatio }
        },
        vertexShader,
        fragmentShader,
        side: THREE.DoubleSide,
        transparent: false,
        depthTest: true,
        depthWrite: true 
    });

    particlesMesh = new THREE.Points(geometry, material);
    scene.add(particlesMesh);

    const maxDim = Math.max(width, height);
    controls.minDistance = 100;
    controls.maxDistance = maxDim * 4;

    if (hasDepth.value) {
        camera.position.set(maxDim * 0.5, maxDim * 0.2, maxDim * 1.2);
    } else {
        camera.position.set(0, 0, maxDim * 1.5);
    }
    controls.target.set(0, 0, 0);
    controls.update();
};

const processImage = (src, isGif = false, file = null, initialDepth = null) => {
    loading.value = true;

    // 如果没有传入初始深度，则重置深度状态；否则保持（后续加载）
    if (!initialDepth) {
        hasDepth.value = false;
        currentDepthTex.value = null;
    }

    if (!isGif) {
        const img = new Image();
        img.crossOrigin = "Anonymous";
        img.src = src;
        img.onload = () => {
            const MAX_WIDTH = 600; // 保持性能限制
            const scale = Math.min(1, MAX_WIDTH / img.width);
            const w = Math.floor(img.width * scale);
            const h = Math.floor(img.height * scale);

            const tempCanvas = document.createElement('canvas');
            tempCanvas.width = w; tempCanvas.height = h;
            const ctx = tempCanvas.getContext('2d');
            ctx.drawImage(img, 0, 0, w, h);

            const texture = new THREE.CanvasTexture(tempCanvas);
            texture.minFilter = THREE.NearestFilter;
            texture.magFilter = THREE.NearestFilter;

            gifState.isGif = false;
            createParticles(w, h, texture);
            loading.value = false;

            if (initialDepth) {
                const loader = new THREE.TextureLoader();
                loader.load(initialDepth, (tex) => {
                    tex.minFilter = THREE.NearestFilter;
                    tex.magFilter = THREE.NearestFilter;
                    currentDepthTex.value = tex;
                    hasDepth.value = true;
                    if (particlesMesh) {
                        particlesMesh.material.uniforms.uDepthTexture.value = tex;
                        particlesMesh.material.uniforms.uHasDepth.value = 1.0;
                        particlesMesh.material.uniforms.uInvertDepth.value = 0.0; // 默认不反转
                        camera.position.set(200, 100, 600); // 初始视角
                        controls.update();
                    }
                });
            }
        };
    } else {
        loadGifFrames(file).then(({ frames, width, height }) => {
            loading.value = false;
        });
    }
};

const handleDepthUpload = (event) => {
    const file = event.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (e) => {
        new THREE.TextureLoader().load(e.target.result, (tex) => {
            tex.minFilter = THREE.NearestFilter;
            tex.magFilter = THREE.NearestFilter;
            currentDepthTex.value = tex;
            hasDepth.value = true;
            invertDepth.value = false;

            if (particlesMesh) {
                particlesMesh.material.uniforms.uDepthTexture.value = tex;
                particlesMesh.material.uniforms.uHasDepth.value = 1.0;
                particlesMesh.material.uniforms.uInvertDepth.value = 0.0;
                controls.update();
            }
        });
    };
    reader.readAsDataURL(file);
};

const clearDepth = () => {
    hasDepth.value = false;
    currentDepthTex.value = null;
    invertDepth.value = false;
    if (particlesMesh) {
        particlesMesh.material.uniforms.uHasDepth.value = 0.0;
        camera.position.set(0, 0, 600);
        controls.update();
    }
};

const handleFileUpload = (event) => {
    const file = event.target.files[0];
    if (!file) return;
    if (file.type === 'image/gif') {
        processImage(null, true, file);
    } else {
        const reader = new FileReader();
        reader.onload = (e) => processImage(e.target.result);
        reader.readAsDataURL(file);
    }
};

const animate = () => {
    animationId = requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
};

const onResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
};

onMounted(() => {
    initScene();
    window.addEventListener('resize', onResize);
    processImage(defaultImg, false, null, defaultDepthImg);
    animate();
});

onUnmounted(() => {
    window.removeEventListener('resize', onResize);
    cancelAnimationFrame(animationId);
    if (renderer) renderer.dispose();
});
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;800&display=swap');
@import url('https://fonts.googleapis.com/css2?family=Material+Icons&display=swap');

.scene-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: #050505;
    font-family: 'Inter', sans-serif;
    overflow: hidden;
}

.webgl-canvas {
    display: block;
    width: 100%;
    height: 100%;
    outline: none;
}

.ui-panel {
    position: absolute;
    bottom: 40px;
    left: 40px;
    width: 340px;
    background: rgba(20, 20, 20, 0.7);
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 24px;
    padding: 24px;
    display: flex;
    flex-direction: column;
    gap: 20px;
    z-index: 10;
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.4);
    color: white;
    pointer-events: auto;
}

.panel-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
}

.panel-header h1 {
    font-size: 24px;
    font-weight: 800;
    margin: 0;
    background: linear-gradient(135deg, #fff, #aaa);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    line-height: 1;
}

.status-indicator {
    font-size: 10px;
    font-weight: 600;
    color: rgba(255, 255, 255, 0.5);
    display: flex;
    align-items: center;
    gap: 6px;
    border: 1px solid rgba(255, 255, 255, 0.1);
    padding: 4px 8px;
    border-radius: 20px;
}

.dot {
    width: 6px;
    height: 6px;
    background: #444;
    border-radius: 50%;
    transition: all 0.3s;
}

.dot.active {
    background: #00ff88;
    box-shadow: 0 0 8px #00ff88;
}

.divider {
    height: 1px;
    background: rgba(255, 255, 255, 0.08);
    width: 100%;
}

.controls-group {
    display: flex;
    flex-direction: column;
    gap: 15px;
}

.control-item {
    display: flex;
    flex-direction: column;
    gap: 8px;
    position: relative;
}

.label-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.label-text {
    font-size: 9px;
    color: rgba(255, 255, 255, 0.4);
    font-weight: 700;
    letter-spacing: 1px;
}

.badge {
    font-size: 8px;
    background: #00ff88;
    color: #000;
    padding: 2px 6px;
    border-radius: 4px;
    font-weight: 800;
}

input[type="file"] {
    display: none;
}

.upload-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
    width: 100%;
    padding: 12px;
    border-radius: 12px;
    font-weight: 600;
    font-size: 12px;
    cursor: pointer;
    transition: all 0.2s;
    box-sizing: border-box;
}

.main-btn {
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: white;
}

.main-btn:hover {
    background: rgba(255, 255, 255, 0.2);
}

.sub-btn {
    background: transparent;
    border: 1px dashed rgba(255, 255, 255, 0.2);
    color: rgba(255, 255, 255, 0.6);
}

.sub-btn:hover {
    border-color: rgba(255, 255, 255, 0.5);
    color: white;
}

.sub-btn.has-file {
    border-style: solid;
    border-color: #00ff88;
    color: #00ff88;
    background: rgba(0, 255, 136, 0.05);
}

.depth-tools {
    position: absolute;
    right: 0;
    top: 28px;
    display: flex;
    gap: 8px;
}

.tool-btn {
    width: 28px;
    height: 28px;
    background: rgba(255, 255, 255, 0.1);
    border-radius: 8px;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 2px;
    cursor: pointer;
    transition: all 0.2s;
    border: 1px solid rgba(255, 255, 255, 0.1);
}

.tool-btn:hover {
    background: rgba(255, 255, 255, 0.2);
    border-color: rgba(255, 255, 255, 0.3);
}

.tool-btn.active {
    background: #6e8efb;
    color: white;
    border-color: #6e8efb;
}

.tool-btn.remove {
    background: rgba(255, 0, 0, 0.15);
    border-color: rgba(255, 0, 0, 0.2);
}

.tool-btn.remove:hover {
    background: rgba(255, 0, 0, 0.4);
}

.tool-text {
    font-size: 8px;
    font-weight: 800;
}

.icon {
    font-family: 'Material Icons';
    font-size: 16px;
}

.icon.small {
    font-size: 14px;
}

.hint {
    font-size: 10px;
    color: rgba(150, 117, 117, 0.3);
    text-align: center;
    margin-top: 5px;
    letter-spacing: 1px;
}
</style>