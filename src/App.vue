<script setup>
import { ref, watch, nextTick } from 'vue'

const fileInput = ref(null)
const image = ref(null)
const imageName = ref('')
const imageSize = ref('')
const pixelSize = ref(12)
const colorMode = ref('original')
const isDragover = ref(false)

const originalCanvas = ref(null)
const pixelCanvas = ref(null)

function handleFile(e) {
  const file = e.target.files?.[0] || e.dataTransfer?.files?.[0]
  if (!file) return
  imageName.value = file.name
  imageSize.value = formatSize(file.size)
  const reader = new FileReader()
  reader.onload = (ev) => {
    const img = new Image()
    img.onload = () => { image.value = img; nextTick(() => draw()) }
    img.src = ev.target.result
  }
  reader.readAsDataURL(file)
}

function formatSize(bytes) {
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB'
}

function onDragOver(e) { e.preventDefault(); isDragover.value = true }
function onDragLeave() { isDragover.value = false }
function onDrop(e) { e.preventDefault(); isDragover.value = false; handleFile(e) }

function clearImage() {
  image.value = null
  imageName.value = ''
  imageSize.value = ''
  if (fileInput.value) fileInput.value.value = ''
}

function draw() {
  if (!image.value) return
  const img = image.value
  const pw = Math.max(2, pixelSize.value)

  // draw original
  const oc = originalCanvas.value
  if (oc) {
    oc.width = img.width
    oc.height = img.height
    const ctx = oc.getContext('2d')
    ctx.drawImage(img, 0, 0)
  }

  // draw pixelated
  const pc = pixelCanvas.value
  if (!pc) return

  const smallW = Math.max(1, Math.floor(img.width / pw))
  const smallH = Math.max(1, Math.floor(img.height / pw))

  // step 1: draw small
  const small = document.createElement('canvas')
  small.width = smallW
  small.height = smallH
  const sctx = small.getContext('2d')
  sctx.drawImage(img, 0, 0, smallW, smallH)

  // step 2: if reduced palette, quantize colors
  if (colorMode.value === 'retro') {
    const data = sctx.getImageData(0, 0, smallW, smallH)
    for (let i = 0; i < data.data.length; i += 4) {
      data.data[i]     = quantize(data.data[i])
      data.data[i + 1] = quantize(data.data[i + 1])
      data.data[i + 2] = quantize(data.data[i + 2])
    }
    sctx.putImageData(data, 0, 0)
  } else if (colorMode.value === 'bw') {
    const data = sctx.getImageData(0, 0, smallW, smallH)
    for (let i = 0; i < data.data.length; i += 4) {
      const gray = (data.data[i] + data.data[i + 1] + data.data[i + 2]) / 3
      const v = gray > 128 ? 255 : 0
      data.data[i] = data.data[i + 1] = data.data[i + 2] = v
    }
    sctx.putImageData(data, 0, 0)
  }

  // step 3: scale up with nearest-neighbor (pixelated look)
  pc.width = img.width
  pc.height = img.height
  const pctx = pc.getContext('2d')
  pctx.imageSmoothingEnabled = false
  pctx.drawImage(small, 0, 0, pc.width, pc.height)
}

function quantize(v) {
  const levels = [0, 51, 102, 153, 204, 255]
  let best = 0, bestDist = Infinity
  for (const l of levels) {
    const d = Math.abs(v - l)
    if (d < bestDist) { bestDist = d; best = l }
  }
  return best
}

function download() {
  if (!pixelCanvas.value) return
  const link = document.createElement('a')
  link.download = 'pixel-art.png'
  link.href = pixelCanvas.value.toDataURL('image/png')
  link.click()
}

watch([pixelSize, colorMode], () => draw())
</script>

<template>
  <header class="app-header">
    <h1>🎨 像素画生成器</h1>
    <p>上传图片 · 拖动滑块 · 一键下载像素风格作品</p>
  </header>

  <!-- upload -->
  <div
    class="upload-zone"
    :class="{ 'has-image': image }"
    @click="() => { if (!image) fileInput?.click() }"
    @dragover="onDragOver"
    @dragleave="onDragLeave"
    @drop="onDrop"
  >
    <template v-if="!image">
      <div class="upload-icon">📁</div>
      <p>拖拽图片到此处，或点击选择文件</p>
      <p class="hint">支持 JPG / PNG / WebP / GIF</p>
    </template>
    <template v-else>
      <div class="file-info">
        <span class="name">{{ imageName }}</span>
        <span class="size">{{ imageSize }}</span>
        <button @click.stop="() => { fileInput?.click() }">换图</button>
        <button @click.stop="clearImage">清除</button>
      </div>
    </template>
    <input ref="fileInput" type="file" accept="image/*" @change="handleFile" />
  </div>

  <!-- controls -->
  <div class="controls" v-if="image">
    <div class="control-group">
      <label>像素大小 <span class="val">{{ pixelSize }}px</span></label>
      <input type="range" v-model.number="pixelSize" min="3" max="48" step="1" />
    </div>
    <div class="control-group">
      <label>色彩模式</label>
      <select v-model="colorMode">
        <option value="original">原始色彩</option>
        <option value="retro">复古 6 色</option>
        <option value="bw">黑白双色</option>
      </select>
    </div>
    <button class="primary" @click="download">⬇ 下载 PNG</button>
  </div>

  <!-- preview -->
  <div class="preview-area" v-if="image">
    <div class="preview-panel">
      <div class="panel-header">
        <span>原图</span>
        <span>{{ image.width }} × {{ image.height }}</span>
      </div>
      <div class="panel-body">
        <canvas ref="originalCanvas" class="smooth"></canvas>
      </div>
    </div>
    <div class="preview-panel">
      <div class="panel-header">
        <span>像素化</span>
        <span>像素 {{ pixelSize }}px · {{ colorMode === 'original' ? '原始色' : colorMode === 'retro' ? '复古色' : '黑白' }}</span>
      </div>
      <div class="panel-body">
        <canvas ref="pixelCanvas"></canvas>
      </div>
    </div>
  </div>

  <div v-else class="empty-state" style="text-align:center;padding:2rem;color:#64748b;">
    上传图片后，这里将显示原图和像素化的对比效果
  </div>

  <footer class="app-footer">
    Made with Vue 3 — <a href="https://github.com/gxw072423/pixel-art">GitHub</a>
  </footer>
</template>

<style scoped>
.app-header { text-align: center; margin-bottom: 2rem; }
.app-header h1 { font-size: 1.8rem; font-weight: 700; color: #f1f5f9; margin-bottom: 0.3rem; }
.app-header p { color: #94a3b8; font-size: 0.9rem; }

.upload-zone {
  border: 2px dashed #334155; border-radius: 12px; padding: 3rem 2rem;
  text-align: center; cursor: pointer;
  transition: border-color 0.2s, background 0.2s; margin-bottom: 1.5rem;
}
.upload-zone:hover { border-color: #3b82f6; background: rgba(59,130,246,0.05); }
.upload-zone.has-image { padding: 1.2rem 1.5rem; border-style: solid; border-color: #334155; }
.upload-icon { font-size: 3rem; margin-bottom: 0.8rem; opacity: 0.6; }
.upload-zone p { color: #94a3b8; font-size: 0.9rem; }
.upload-zone .hint { font-size: 0.78rem; color: #64748b; margin-top: 0.4rem; }
.upload-zone input[type="file"] { display: none; }

.file-info { display: flex; align-items: center; gap: 0.8rem; justify-content: center; }
.file-info .name { color: #e2e8f0; font-weight: 600; font-size: 0.9rem; }
.file-info .size { color: #64748b; font-size: 0.78rem; }
.file-info button {
  background: #334155; color: #94a3b8; border: none;
  padding: 0.35rem 0.8rem; border-radius: 6px; cursor: pointer;
  font-size: 0.8rem; transition: background 0.2s;
}
.file-info button:hover { background: #475569; color: #e2e8f0; }

.controls {
  display: flex; gap: 1.5rem; flex-wrap: wrap; align-items: flex-end;
  margin-bottom: 1.5rem; padding: 1.2rem 1.5rem;
  background: #1e293b; border-radius: 10px; border: 1px solid #334155;
}
.control-group { display: flex; flex-direction: column; gap: 0.3rem; }
.control-group label { font-size: 0.78rem; color: #94a3b8; font-weight: 500; }
.control-group .val { color: #3b82f6; font-weight: 600; }
.control-group input[type="range"] { width: 140px; accent-color: #3b82f6; }
.control-group select {
  background: #0f172a; color: #e2e8f0; border: 1px solid #334155;
  border-radius: 6px; padding: 0.4rem 0.7rem; font-size: 0.85rem; cursor: pointer;
}
button.primary {
  background: #3b82f6; color: #fff; border: none; padding: 0.55rem 1.4rem;
  border-radius: 8px; font-size: 0.9rem; font-weight: 600;
  cursor: pointer; transition: background 0.2s; white-space: nowrap;
}
button.primary:hover { background: #2563eb; }

.preview-area {
  display: grid; grid-template-columns: 1fr 1fr; gap: 1.5rem;
}
@media (max-width: 700px) { .preview-area { grid-template-columns: 1fr; } .controls { gap: 1rem; } }

.preview-panel { background: #1e293b; border-radius: 10px; border: 1px solid #334155; overflow: hidden; }
.preview-panel :deep(.panel-header) {
  padding: 0.7rem 1rem; font-size: 0.78rem; color: #94a3b8;
  font-weight: 600; border-bottom: 1px solid #334155;
  display: flex; justify-content: space-between; align-items: center;
}
.preview-panel :deep(.panel-body) {
  padding: 1.5rem; display: flex; align-items: center; justify-content: center;
  min-height: 280px;
  background: repeating-conic-gradient(#1a2332 0% 25%, #1e2a3a 0% 50%) 50% / 20px 20px;
}
.preview-panel :deep(canvas) { max-width: 100%; max-height: 400px; border-radius: 4px; }
.preview-panel :deep(canvas:not(.smooth)) { image-rendering: pixelated; }

.app-footer { text-align: center; margin-top: 2.5rem; color: #475569; font-size: 0.75rem; }
.app-footer a { color: #64748b; }

.empty-state { text-align: center; padding: 2rem; color: #64748b; }
</style>
