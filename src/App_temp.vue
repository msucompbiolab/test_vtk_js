

<template>
  <div>
    <header>
      <h1>STL Viewer (vtk.js)</h1>
      <input type="file" accept=".stl" @change="onFileChange" />
      <span style="margin-left:12px">{{ status }}</span>
    </header>

    <section>
      <div ref="vtkContainer" class="vtk-container"></div>
    </section>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'

import vtkRenderWindow from '@kitware/vtk.js/Rendering/Core/RenderWindow'
import vtkRenderer from '@kitware/vtk.js/Rendering/Core/Renderer'
import vtkOpenGLRenderWindow from '@kitware/vtk.js/Rendering/OpenGL/RenderWindow'
import vtkRenderWindowInteractor from '@kitware/vtk.js/Rendering/Core/RenderWindowInteractor'
import vtkInteractorStyleTrackballCamera from '@kitware/vtk.js/Interaction/Style/InteractorStyleTrackballCamera'
import vtkSTLReader from '@kitware/vtk.js/IO/Geometry/STLReader'
import vtkMapper from '@kitware/vtk.js/Rendering/Core/Mapper'
import vtkActor from '@kitware/vtk.js/Rendering/Core/Actor'

const vtkContainer = ref(null)
const status = ref('No file loaded')

let fullScreenRenderWindow = null
let renderer = null
let renderWindow = null
let glwindow = null
let interactor = null

onMounted(() => {
  // Try to initialize a manual render pipeline now
  try {
    ensureRenderSetup()
  } catch (e) {
    console.error('ensureRenderSetup failed in onMounted:', e)
  }
})

onBeforeUnmount(() => {
  try {
    if (interactor) {
      interactor.releaseEvents && interactor.releaseEvents()
      interactor = null
    }
  } catch (e) {
    console.warn('Error releasing interactor:', e)
  }

  try {
    if (renderWindow) {
      renderWindow.delete && renderWindow.delete()
      renderWindow = null
    }
  } catch (e) {
    console.warn('Error deleting renderWindow:', e)
  }

  try {
    if (glwindow) {
      glwindow.delete && glwindow.delete()
      glwindow = null
    }
  } catch (e) {
    console.warn('Error deleting glwindow:', e)
  }
})

function clearScene() {
  if (renderer) {
    renderer.removeAllViewProps()
    renderWindow.render()
  }
}

function ensureRenderSetup() {
  console.log('ensureRenderSetup (manual): vtkContainer=', vtkContainer.value)
  const el = vtkContainer.value || document.querySelector('.vtk-container')
  if (!el) {
    console.warn('No container element found for manual render setup')
    return false
  }

  try {
    // If already initialized, ensure objects exist
    if (renderer && renderWindow && glwindow) return true

    // create or find canvas inside container
    let canvas = el.querySelector('canvas')
    if (!canvas) {
      canvas = document.createElement('canvas')
      canvas.style.width = '100%'
      canvas.style.height = '100%'
      canvas.style.display = 'block'
      el.appendChild(canvas)
    }

    renderWindow = vtkRenderWindow.newInstance()
    glwindow = vtkOpenGLRenderWindow.newInstance()
    glwindow.setContainer(canvas)
    renderWindow.addView(glwindow)

    renderer = vtkRenderer.newInstance()
    renderWindow.addRenderer(renderer)

    interactor = vtkRenderWindowInteractor.newInstance()
    // set the view to the API-specific render window (OpenGL) not the renderWindow
    interactor.setView(glwindow)
    interactor.initialize()
    interactor.bindEvents(canvas)
    interactor.setInteractorStyle(vtkInteractorStyleTrackballCamera.newInstance())

    console.log('Manual vtk setup complete', { renderWindow, renderer, glwindow, interactor })
    return true
  } catch (e) {
    console.error('Manual render setup failed:', e)
    return false
  }
}

async function onFileChange(e) {
  const files = e.target.files
  if (!files || files.length !== 1) {
    status.value = 'Please select a single .stl file'
    return
  }

  const file = files[0]
  status.value = `Reading ${file.name}...`

  // ensure DOM updates and refs are ready
  await nextTick()

  const reader = new FileReader()
  reader.onload = () => {
    try {
      // Ensure renderer/renderWindow are available (avoid null addActor)
      if (!ensureRenderSetup()) {
        console.error('ensureRenderSetup failed; vtkContainer:', vtkContainer.value, 'document.querySelector:', document.querySelector('.vtk-container'))
        throw new Error('Renderer not initialized')
      }

      const arrayBuffer = reader.result

      const stlReader = vtkSTLReader.newInstance()
      // Try parsing with the ArrayBuffer-specific API, fallback to generic parse()
      try {
        stlReader.parseAsArrayBuffer(arrayBuffer)
      } catch (parseErr) {
        console.warn('parseAsArrayBuffer failed, trying parse():', parseErr)
        // parse accepts String or ArrayBuffer
        stlReader.parse(arrayBuffer)
      }

      const polydata = stlReader.getOutputData(0)

      // Basic sanity checks
      if (!polydata) {
        throw new Error('No polydata returned by STL reader')
      }
      const pts = polydata.getPoints && polydata.getPoints()
      const nPts = pts ? pts.getNumberOfPoints() : 0
      if (!nPts) {
        throw new Error('STL parsed but contains 0 points')
      }

      clearScene()

      const mapper = vtkMapper.newInstance()
      mapper.setInputData(polydata)

      const actor = vtkActor.newInstance()
      actor.setMapper(mapper)

      renderer.addActor(actor)
      renderer.resetCamera()
      renderWindow.render()

      status.value = `Loaded: ${file.name}`
    } catch (err) {
      console.error('STL load error:', err)
      // Show the error message in the UI to help debugging
      status.value = `Failed to load STL: ${err && err.message ? err.message : err}`
    }
  }

  reader.onerror = (err) => {
    console.error(err)
    status.value = 'Error reading file'
  }

  reader.readAsArrayBuffer(file)
}
</script>

<style>
.vtk-container {
  width: 100%;
  height: 600px;
  border: 1px solid #333;
}
header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
}

/* Ensure the header and file input are above the vtk.js canvas */
header {
  position: relative;
  z-index: 1000;
}

input[type="file"] {
  position: relative;
  z-index: 1001;
}

.vtk-container {
  position: relative;
  z-index: 0;
}
</style>
