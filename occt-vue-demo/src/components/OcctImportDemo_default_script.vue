<template>
  <div class="occt-import-demo">
    <h1>OCCT Import JS Demo</h1>
    <p>Upload a .step, .iges, or .brep file to see its JSON structure and 3D visualization.</p>

    <div class="input-section">
      <input type="file" @change="handleFileUpload" accept=".step,.stp,.iges,.brep" />
      <button @click="processFile" :disabled="!selectedFile || loading">
        {{ loading ? 'Processing...' : 'Process File' }}
      </button>
    </div>

    <div v-if="loading" class="status-message">Loading and processing... Please wait.</div>
    <div v-if="error" class="error-message">
      <h2>Error:</h2>
      <p>{{ error }}</p>
    </div>

    <div class="viewer-section" v-if="result && result.success">
      <h2>3D Viewer</h2>
      <canvas ref="threeCanvas" class="three-canvas"></canvas>
      <div v-if="result.success" class="success-message">
        <p>Import successful! You can interact with the 3D model above.</p>
      </div>
    </div>

    <div v-if="result" class="result-display">
      <h2>Import Result (JSON):</h2>
      <pre>{{ JSON.stringify(result, null, 2) }}</pre>
    </div>
  </div>
</template>

<script>
import occtimportjs from 'occt-import-js';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

export default {
  name: 'OcctImportDemo',
  data() {
    return {
      selectedFile: null,
      fileContent: null, // 存储文件内容为 Uint8Array
      loading: false,
      result: null,
      error: null,
      // IMPORTANT: Three.js 相关的属性（scene, camera, renderer, controls, modelGroup）
      // 已从 data() 中移除，以避免 Vue 的响应式处理。
      // 它们将在 initThreeDScene() 中直接作为 this 的属性进行初始化。
      animationFrameId: null, // 这个保持在 data() 中，因为它是 Vue 生命周期相关的。
    };
  },
  mounted() {
    // 组件挂载后初始化 Three.js 场景。
    // $nextTick 确保 canvas 元素在初始化时已渲染完成。
    this.$nextTick(() => {
      this.initThreeDScene();
    });
    // 监听窗口大小变化，以调整 Three.js 渲染器。
    window.addEventListener('resize', this.onWindowResize);
  },
  beforeUnmount() {
    // 组件销毁前清理 Three.js 资源，防止内存泄漏。
    if (this.animationFrameId) {
      cancelAnimationFrame(this.animationFrameId);
      this.animationFrameId = null; // 清理引用
    }
    if (this.renderer) {
      this.renderer.dispose(); // 释放 GPU 资源
      this.renderer.domElement = null; // 移除 canvas 引用
      this.renderer = null; // 清理引用
    }
    if (this.controls) {
      this.controls.dispose(); // 释放控制器资源
      this.controls = null; // 清理引用
    }
    if (this.scene) {
      // 遍历场景中的所有对象，销毁它们的几何体和材质。
      this.scene.traverse((object) => {
        if (object.isMesh) {
          if (object.geometry) object.geometry.dispose();
          if (object.material) {
            if (Array.isArray(object.material)) {
              object.material.forEach(material => material.dispose());
            } else {
              object.material.dispose();
            }
          }
        }
      });
      this.scene = null; // 清理引用
    }
    if (this.camera) {
        this.camera = null; // 清理引用
    }
    if (this.modelGroup) {
        this.modelGroup = null; // 清理引用
    }
    // 移除窗口大小变化监听器。
    window.removeEventListener('resize', this.onWindowResize);
  },
  methods: {
    /**
     * 处理文件输入变化事件，读取选中的文件内容为 Uint8Array。
     * @param {Event} event 文件输入变化事件。
     */
    handleFileUpload(event) {
      this.selectedFile = event.target.files[0];
      this.result = null; // 清除之前的导入结果
      this.error = null;   // 清除之前的错误信息
      this.fileContent = null; // 清除之前的文件内容
      this.clearModel(); // 清除任何已显示的模型

      if (this.selectedFile) {
        const reader = new FileReader();
        reader.onload = (e) => {
          this.fileContent = new Uint8Array(e.target.result);
        };
        reader.onerror = (e) => {
          this.error = 'Failed to read file: ' + e.target.error;
          this.fileContent = null;
        };
        reader.readAsArrayBuffer(this.selectedFile);
      }
    },

    /**
     * 使用 occt-import-js 处理上传的文件。
     */
    async processFile() {
      if (!this.fileContent) {
        this.error = 'No file selected or file content not loaded.';
        return;
      }

      this.loading = true;
      this.result = null;
      this.error = null;
      this.clearModel(); // 在导入新文件前清除现有模型

      try {
        // 初始化 Emscripten 模块。
        // 需要指定 .wasm 文件的位置。假设 occt-import-js.wasm 位于 public 文件夹中。
        const occt = await occtimportjs({
          locateFile: (path, prefix) => {
            if (path.endsWith('.wasm')) {
              return `/${path}`; // 假设 .wasm 文件在服务器根目录
            }
            return prefix + path; // 其他文件路径的回退
          }
        });

        let importFunction;
        const fileName = this.selectedFile.name.toLowerCase();

        // 根据文件扩展名确定正确的导入函数。
        if (fileName.endsWith('.step') || fileName.endsWith('.stp')) {
          importFunction = occt.ReadStepFile;
        } else if (fileName.endsWith('.iges') || fileName.endsWith('.igs')) {
          importFunction = occt.ReadIgesFile;
        } else if (fileName.endsWith('.brep')) {
          importFunction = occt.ReadBrepFile;
        } else {
          this.error = 'Unsupported file type. Please select .step, .iges, or .brep.';
          this.loading = false;
          return;
        }

        // 示例三角化参数。可设为 null 使用默认值。
        const params = {
          linearUnit: 'millimeter', // 输出线性单位
          linearDeflectionType: 'bounding_box_ratio', // 线性偏差类型 (相对于包围盒的比例)
          linearDeflection: 0.1, // 线性偏差值 (10% 的包围盒大小)
          angularDeflection: 0.5 // 角度偏差 (弧度，约 28.6 度)
        };

        console.log('Starting file import...');
        const importResult = importFunction(this.fileContent, params);
        console.log('Import result:', importResult);

        this.result = importResult;
        if (!importResult.success) {
          this.error = 'Import failed: ' + (importResult.error || 'Unknown error occurred during import.');
        } else {
          // 导入成功后，在 Three.js 中渲染模型。
          this.$nextTick(() => { // 确保 canvas 已渲染，再更新 3D 视图
            this.renderModel(this.result);
          });
        }
      } catch (e) {
        this.error = 'Error during OCCT import initialization or processing: ' + e.message;
        console.error('OCCT Import Error:', e);
      } finally {
        this.loading = false;
      }
    },

    /**
     * 初始化 Three.js 场景、相机、渲染器和控制器。
     * 此方法在组件挂载时调用一次。
     */
    initThreeDScene() {
      const canvas = this.$refs.threeCanvas;
      if (!canvas) {
        // 如果 canvas 元素在 mounted 时未立即可用（例如，由于 v-if），
        // 则在 renderModel 被 $nextTick 调用时会再次尝试初始化。
        return;
      }

      // IMPORTANT: Three.js 相关的对象现在直接作为 'this' 的属性进行初始化。
      // Vue 不会尝试将它们转换为响应式对象，这解决了 'modelViewMatrix' 错误。
      this.scene = new THREE.Scene();
      this.scene.background = new THREE.Color(0xf0f0f0); // 浅灰色背景

      this.camera = new THREE.PerspectiveCamera(75, canvas.clientWidth / canvas.clientHeight, 0.1, 1000);
      this.camera.position.set(5, 5, 5); // 初始摄像机位置

      this.renderer = new THREE.WebGLRenderer({ canvas: canvas, antialias: true });
      this.renderer.setPixelRatio(window.devicePixelRatio); // 处理高 DPI 屏幕
      this.renderer.setSize(canvas.clientWidth, canvas.clientHeight);

      // 灯光
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5); // 柔和的白光
      this.scene.add(ambientLight);
      const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
      directionalLight.position.set(1, 1, 1).normalize(); // 来自右上方前方的光
      this.scene.add(directionalLight);

      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      this.controls.enableDamping = true; // 启用阻尼效果，实现平滑移动
      this.controls.dampingFactor = 0.25;
      this.controls.screenSpacePanning = false; // 禁止屏幕空间平移
      this.controls.minDistance = 1; // 最小缩放距离
      this.controls.maxDistance = 500; // 最大缩放距离

      // 模型组：所有导入的网格都将添加到这个组中，方便管理和清除。
      this.modelGroup = new THREE.Group();
      this.scene.add(this.modelGroup);

      // 启动动画循环。
      this.animate();
    },

    /**
     * 处理窗口大小变化事件，调整渲染器和摄像机。
     */
    onWindowResize() {
      const canvas = this.$refs.threeCanvas;
      if (canvas && this.camera && this.renderer) {
        // 获取父元素的宽度，使 canvas 响应式。
        const parent = canvas.parentElement;
        const width = parent.clientWidth;
        // 保持宽高比或设置最大高度。
        const height = Math.min(width * 0.75, 500); // 最大高度设置为 500px

        this.camera.aspect = width / height;
        this.camera.updateProjectionMatrix(); // 更新摄像机投影矩阵
        this.renderer.setSize(width, height);
      }
    },

    /**
     * Three.js 动画循环。
     */
    animate() {
      this.animationFrameId = requestAnimationFrame(this.animate); // 请求下一帧动画
      if (this.controls) this.controls.update(); // 更新控制器（启用阻尼时需要）
      if (this.renderer && this.scene && this.camera) {
        this.renderer.render(this.scene, this.camera); // 渲染场景
      }
    },

    /**
     * 清除 modelGroup 中的所有现有网格。
     * 正确地处理 Three.js 几何体和材质的 dispose()，以防止内存泄漏。
     */
    clearModel() {
      if (this.modelGroup) {
        // 遍历并移除组中的所有子对象。
        while (this.modelGroup.children.length > 0) {
          const object = this.modelGroup.children[0];
          this.modelGroup.remove(object); // 从组中移除

          // 释放几何体和材质以释放 GPU 内存。
          if (object.geometry) {
            object.geometry.dispose();
          }
          if (object.material) {
            if (Array.isArray(object.material)) {
              object.material.forEach(material => material.dispose());
            } else {
              object.material.dispose();
            }
          }
        }
      }
    },

    /**
     * 将 occt-import-js 导入结果中的模型渲染到 Three.js 场景中。
     * @param {Object} importResult occt-import-js 的 JSON 结果。
     */
    renderModel(importResult) {
      // 确保 Three.js 场景已初始化，尤其是在 canvas 刚可用时。
      if (!this.scene || !this.camera || !this.renderer) {
        this.initThreeDScene();
        if (!this.scene) {
            console.error('Three.js scene still not initialized after attempt. Cannot render model.');
            return;
        }
      }

      this.clearModel(); // 渲染新模型前总是清除旧模型。

      if (!importResult || !importResult.meshes || importResult.meshes.length === 0) {
        console.warn('No meshes found in import result to render.');
        this.resetCameraView(); // 如果没有网格，则重置摄像机视图。
        return;
      }

      const bbox = new THREE.Box3();
      let firstMesh = true;

      importResult.meshes.forEach(meshData => {
        // --- 1. 数据完整性检查 ---
        // 确保 position 数据存在且有效。
        if (!meshData.attributes || !meshData.attributes.position || !meshData.attributes.position.array) {
            console.warn(`Mesh "${meshData.name || 'Unnamed Mesh'}" is missing valid position data. Skipping this mesh.`);
            return; // 跳过当前网格，处理下一个。
        }

        // 确保 index 数据存在且有效。
        // 关键修正：根据库的文档，'index' 直接是 'meshData' 的属性，而不是 'attributes' 的子属性。
        if (!meshData.index || !meshData.index.array) {
            console.warn(`Mesh "${meshData.name || 'Unnamed Mesh'}" is missing valid index data. Skipping this mesh.`);
            return; // 跳过当前网格，处理下一个。
        }

        const geometry = new THREE.BufferGeometry();

        // --- 2. 处理位置数据 ---
        const positions = new Float32Array(meshData.attributes.position.array);
        geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

        // --- 3. 处理法线数据 (可选) ---
        if (meshData.attributes.normal && meshData.attributes.normal.array) {
          const normals = new Float32Array(meshData.attributes.normal.array);
          geometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3));
        } else {
            console.warn(`Mesh "${meshData.name || 'Unnamed Mesh'}" has no normals. Computing them, which might be less efficient.`);
            geometry.computeVertexNormals(); // 如果没有提供法线，Three.js 可以尝试计算。
        }

        // --- 4. 处理索引数据 ---
        // 如果顶点数量非常大 (> 65535 个顶点)，则使用 Uint32Array。
        const indices = new (positions.length / 3 > 65535 ? Uint32Array : Uint16Array)(meshData.index.array);
        geometry.setIndex(new THREE.BufferAttribute(indices, 1));

        // --- 5. 创建材质 ---
        let material;
        if (meshData.color && meshData.color.length === 3) {
          const color = new THREE.Color(
            meshData.color[0] / 255,
            meshData.color[1] / 255,
            meshData.color[2] / 255
          );
          material = new THREE.MeshStandardMaterial({ color: color, side: THREE.DoubleSide });
        } else {
          material = new THREE.MeshStandardMaterial({ color: 0xcccccc, side: THREE.DoubleSide }); // 默认灰色材质
        }

        // --- 6. 创建网格并添加到场景 ---
        const mesh = new THREE.Mesh(geometry, material);
        this.modelGroup.add(mesh); // 添加到模型组中，便于管理。

        // --- 7. 更新整体包围盒 (用于摄像机适配) ---
        if (firstMesh) {
          bbox.setFromObject(mesh);
          firstMesh = false;
        } else {
          bbox.union(new THREE.Box3().setFromObject(mesh));
        }
      });

      // --- 8. 适配摄像机到模型 ---
      if (!bbox.isEmpty()) {
        const center = new THREE.Vector3();
        const size = new THREE.Vector3();
        bbox.getCenter(center);
        bbox.getSize(size);

        const maxDim = Math.max(size.x, size.y, size.z);
        const fov = this.camera.fov * (Math.PI / 180); // 将 FOV 从度转换为弧度
        let cameraZ = Math.abs(maxDim / 2 / Math.tan(fov / 2)); // 计算所需的摄像机距离

        // 根据宽高比调整摄像机距离，确保模型在水平方向也能完全显示。
        cameraZ *= (this.camera.aspect > 1 ? this.camera.aspect : 1);

        this.camera.position.copy(center); // 将摄像机移动到模型的中心
        this.camera.position.z += cameraZ;
        this.camera.position.y += cameraZ * 0.5; // 稍微抬高摄像机，获得更好的视角

        this.camera.lookAt(center); // 让摄像机看向模型的中心

        if (this.controls) {
            this.controls.target.copy(center); // 让控制器围绕模型的中心进行轨道运动。
            this.controls.update();
        }
      } else {
         // 如果模型为空，则重置摄像机到默认位置。
         this.resetCameraView();
      }
    },

    /**
     * 将摄像机重置到默认的查看位置。
     */
    resetCameraView() {
      if (this.camera) {
        this.camera.position.set(5, 5, 5);
        this.camera.lookAt(new THREE.Vector3(0,0,0));
      }
      if (this.controls) {
        this.controls.target.set(0,0,0);
        this.controls.update();
      }
    }
  },
};
</script>

<style scoped>
.occt-import-demo {
  max-width: 900px; /* Slightly wider to accommodate viewer */
  margin: 40px auto;
  padding: 20px;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  text-align: center;
}

h1 {
  color: #333;
  margin-bottom: 20px;
}

.input-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px; /* Spacing between input and button */
  margin-bottom: 20px;
}

input[type="file"] {
  display: block;
  width: fit-content;
  margin-left: auto;
  margin-right: auto;
}

button {
  background-color: #42b983;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s ease;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background-color: #369c73;
}

.status-message {
  margin-top: 20px;
  color: #007bff;
  font-style: italic;
}

.error-message {
  margin-top: 20px;
  color: #dc3545;
  background-color: #f8d7da;
  border: 1px solid #f5c6cb;
  padding: 10px;
  border-radius: 5px;
}

/* New styles for 3D Viewer */
.viewer-section {
  margin-top: 30px;
  padding-top: 20px;
  border-top: 1px solid #e0e0e0; /* Separator line */
}

.viewer-section h2 {
  color: #555;
  margin-bottom: 15px;
}

.three-canvas {
  width: 100%; /* Makes canvas fill its parent's width */
  height: 500px; /* Set a fixed height for the canvas */
  display: block; /* Remove extra space below the canvas */
  background-color: #f0f0f0; /* Matches scene background for consistency */
  border: 1px solid #ddd;
  border-radius: 5px;
  box-shadow: 0 0 8px rgba(0, 0, 0, 0.1); /* Subtle shadow for depth */
}


.result-display {
  margin-top: 20px;
  text-align: left;
}

.result-display h2 {
  color: #555;
  margin-bottom: 10px;
}

.result-display pre {
  background-color: #f8f8f8;
  padding: 15px;
  border: 1px solid #ddd;
  border-radius: 5px;
  overflow-x: auto;
  white-space: pre-wrap; /* Ensures long lines wrap */
  word-wrap: break-word; /* Breaks long words */
  max-height: 400px; /* Limit height to prevent excessive scrolling */
}

.success-message {
  margin-top: 10px;
  color: #28a745;
  font-weight: bold;
}
</style>