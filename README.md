# OCCT + Three.js STEP Viewer 🌐🔧

A simple web-based 3D viewer for `.step`, `.iges`, and `.brep` CAD files using [OCCT Import JS](https://github.com/shapeformer/occt-import-js) and [Three.js](https://threejs.org/).

> Easily drag and drop CAD files into your browser and explore them in 3D!

---

## ✨ Features

- ✅ Import and visualize `.step`, `.stp`, `.iges`, `.igs`, and `.brep` files.
- ✅ Interactive 3D viewer using Three.js + OrbitControls.
- ✅ JSON-based mesh structure display.
- ✅ File processing powered by WebAssembly OCCT engine.
- ✅ Clean UI with loading and error states.
- ✅ Fully client-side (no server required).


---

## 🚀 Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/occt-step-viewer.git
cd occt-vue-demo
```

2. Install Dependencies
```bash
npm install
```

3. Start Development Server
```bash
npm run dev
```
Open your browser and go to http://localhost:5173 (or the Vite port).


🧪 Supported File Formats
- .step, .stp (STEP)

⚠️ Ensure the .wasm file from occt-import-js is served correctly (placed in /public folder).

🧱 Tech Stack
- Vue 3 + <script setup>	UI Framework
- Three.js	3D Rendering
- OCCT Import JS	CAD parsing via WebAssembly
- Vite	Fast dev build


🛠 Known Issues
Large STEP/IGES files may be slow due to WASM memory limitations.
File processing is done entirely in the browser; no upload occurs.

📄 License
MIT

🙋‍♂️ Author
Made with Roadinforest

⭐️ Star This Project
If you find it useful, feel free to star the repo 🌟 and share it!
