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

<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import occtimportjs from 'occt-import-js'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const selectedFile = ref(null)
const fileContent = ref(null)
const loading = ref(false)
const result = ref(null)
const error = ref(null)
const animationFrameId = ref(null)
const threeCanvas = ref(null)

// 非响应式 Three.js 对象
let scene = null
let camera = null
let renderer = null
let controls = null
let modelGroup = null

function handleFileUpload(event) {
  selectedFile.value = event.target.files[0]
  result.value = null
  error.value = null
  fileContent.value = null
  clearModel()

  if (selectedFile.value) {
    const reader = new FileReader()
    reader.onload = (e) => {
      fileContent.value = new Uint8Array(e.target.result)
    }
    reader.onerror = (e) => {
      error.value = 'Failed to read file: ' + e.target.error
      fileContent.value = null
    }
    reader.readAsArrayBuffer(selectedFile.value)
  }
}

async function processFile() {
  if (!fileContent.value) {
    error.value = 'No file selected or file content not loaded.'
    return
  }

  loading.value = true
  result.value = null
  error.value = null
  clearModel()

  try {
    const occt = await occtimportjs({
      locateFile: (path, prefix) => path.endsWith('.wasm') ? `/${path}` : prefix + path
    })

    const fileName = selectedFile.value.name.toLowerCase()
    let importFunction = null

    if (fileName.endsWith('.step') || fileName.endsWith('.stp')) {
      importFunction = occt.ReadStepFile
    } else if (fileName.endsWith('.iges') || fileName.endsWith('.igs')) {
      importFunction = occt.ReadIgesFile
    } else if (fileName.endsWith('.brep')) {
      importFunction = occt.ReadBrepFile
    } else {
      error.value = 'Unsupported file type. Please select .step, .iges, or .brep.'
      loading.value = false
      return
    }

    const params = {
      linearUnit: 'millimeter',
      linearDeflectionType: 'bounding_box_ratio',
      linearDeflection: 0.1,
      angularDeflection: 0.5
    }

    const importResult = importFunction(fileContent.value, params)
    result.value = importResult

    if (!importResult.success) {
      error.value = 'Import failed: ' + (importResult.error || 'Unknown error.')
    } else {
      await nextTick()
      renderModel(importResult)
    }
  } catch (e) {
    error.value = 'Error during OCCT import: ' + e.message
    console.error(e)
  } finally {
    loading.value = false
  }
}

function initThreeDScene() {
  const canvas = threeCanvas.value
  if (!canvas) return

  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xf0f0f0)

  camera = new THREE.PerspectiveCamera(75, canvas.clientWidth / canvas.clientHeight, 0.1, 1000)
  camera.position.set(5, 5, 5)

  renderer = new THREE.WebGLRenderer({ canvas, antialias: true })
  renderer.setPixelRatio(window.devicePixelRatio)
  renderer.setSize(canvas.clientWidth, canvas.clientHeight)

  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5)
  scene.add(ambientLight)
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
  directionalLight.position.set(1, 1, 1).normalize()
  scene.add(directionalLight)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.25
  controls.screenSpacePanning = false
  controls.minDistance = 1
  controls.maxDistance = 500

  modelGroup = new THREE.Group()
  scene.add(modelGroup)

  animate()
}

function animate() {
  animationFrameId.value = requestAnimationFrame(animate)
  if (controls) controls.update()
  if (renderer && scene && camera) {
    renderer.render(scene, camera)
  }
}

function onWindowResize() {
  const canvas = threeCanvas.value
  if (canvas && camera && renderer) {
    const parent = canvas.parentElement
    const width = parent.clientWidth
    const height = Math.min(width * 0.75, 500)

    camera.aspect = width / height
    camera.updateProjectionMatrix()
    renderer.setSize(width, height)
  }
}

function clearModel() {
  if (modelGroup) {
    while (modelGroup.children.length > 0) {
      const object = modelGroup.children[0]
      modelGroup.remove(object)
      if (object.geometry) object.geometry.dispose()
      if (object.material) {
        if (Array.isArray(object.material)) object.material.forEach(m => m.dispose())
        else object.material.dispose()
      }
    }
  }
}

function renderModel(importResult) {
  if (!scene || !camera || !renderer) {
    initThreeDScene()
    if (!scene) return
  }

  clearModel()

  if (!importResult || !importResult.meshes || importResult.meshes.length === 0) {
    resetCameraView()
    return
  }

  const bbox = new THREE.Box3()
  let firstMesh = true

  importResult.meshes.forEach(meshData => {
    if (!meshData.attributes?.position?.array || !meshData.index?.array) {
      console.warn(`Skipping invalid mesh: ${meshData.name}`)
      return
    }

    const geometry = new THREE.BufferGeometry()
    const positions = new Float32Array(meshData.attributes.position.array)
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3))

    if (meshData.attributes.normal?.array) {
      const normals = new Float32Array(meshData.attributes.normal.array)
      geometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3))
    } else {
      geometry.computeVertexNormals()
    }

    const indices = new (positions.length / 3 > 65535 ? Uint32Array : Uint16Array)(meshData.index.array)
    geometry.setIndex(new THREE.BufferAttribute(indices, 1))

    const material = meshData.color && meshData.color.length === 3
      ? new THREE.MeshStandardMaterial({
          color: new THREE.Color(...meshData.color.map(c => c / 255)),
          side: THREE.DoubleSide
        })
      : new THREE.MeshStandardMaterial({ color: 0xcccccc, side: THREE.DoubleSide })

    const mesh = new THREE.Mesh(geometry, material)
    modelGroup.add(mesh)

    if (firstMesh) {
      bbox.setFromObject(mesh)
      firstMesh = false
    } else {
      bbox.union(new THREE.Box3().setFromObject(mesh))
    }
  })

  if (!bbox.isEmpty()) {
    const center = new THREE.Vector3()
    const size = new THREE.Vector3()
    bbox.getCenter(center)
    bbox.getSize(size)

    const maxDim = Math.max(size.x, size.y, size.z)
    const fov = camera.fov * (Math.PI / 180)
    let cameraZ = Math.abs(maxDim / 2 / Math.tan(fov / 2))
    cameraZ *= (camera.aspect > 1 ? camera.aspect : 1)

    camera.position.copy(center)
    camera.position.z += cameraZ
    camera.position.y += cameraZ * 0.5
    camera.lookAt(center)

    if (controls) {
      controls.target.copy(center)
      controls.update()
    }
  } else {
    resetCameraView()
  }
}

function resetCameraView() {
  if (camera) {
    camera.position.set(5, 5, 5)
    camera.lookAt(new THREE.Vector3(0, 0, 0))
  }
  if (controls) {
    controls.target.set(0, 0, 0)
    controls.update()
  }
}

onMounted(() => {
  nextTick(() => initThreeDScene())
  window.addEventListener('resize', onWindowResize)
})

onBeforeUnmount(() => {
  if (animationFrameId.value) cancelAnimationFrame(animationFrameId.value)
  if (renderer) {
    renderer.dispose()
    renderer.domElement = null
    renderer = null
  }
  if (controls) controls.dispose()
  scene?.traverse(obj => {
    if (obj.isMesh) {
      obj.geometry?.dispose()
      if (Array.isArray(obj.material)) obj.material.forEach(m => m.dispose())
      else obj.material?.dispose()
    }
  })
  scene = camera = modelGroup = null
  window.removeEventListener('resize', onWindowResize)
})
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