<template>
  <div class="app-container">
    <h1 class="title">My VTK Application</h1>
    <div class="main-layout">
      <div ref="vtkContainer" class="vtk-canvas" />
      <div class="info-panel">
        <h2>3D Cone Visualization</h2>
        <p>
          This is an interactive 3D visualization of a cone model using VTK.js.
          The cone is rendered in real-time with adjustable parameters.
        </p>
        <h3>Features:</h3>
        <ul>
          <li>Interactive 3D rotation and zoom</li>
          <li>Real-time rendering updates</li>
          <li>Customizable cone resolution</li>
          <li>Multiple rendering modes</li>
        </ul>
        <h3>How to Use:</h3>
        <ul>
          <li>Drag to rotate the view</li>
          <li>Scroll to zoom in/out</li>
          <li>Use controls to adjust parameters</li>
        </ul>
      </div>
    </div>
    <table class="controls">
      <tbody>
        <tr>
          <td>
            <label>Representation:</label>
          </td>
          <td>
            <select
              :value="representation"
              @change="setRepresentation($event.target.value)"
            >
              <option value="0">Points</option>
              <option value="1">Wireframe</option>
              <option value="2">Surface</option>
            </select>
          </td>
        </tr>
        <tr>
          <td>
            <label>Resolution:</label>
          </td>
          <td>
            <input
              type="range"
              min="4"
              max="80"
              :value="coneResolution"
              @input="setConeResolution($event.target.value)"
            />
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import { ref, unref, onMounted, onBeforeUnmount, watchEffect } from 'vue';

import '@kitware/vtk.js/Rendering/Profiles/Geometry';

import vtkFullScreenRenderWindow from '@kitware/vtk.js/Rendering/Misc/FullScreenRenderWindow';

import vtkActor           from '@kitware/vtk.js/Rendering/Core/Actor';
import vtkMapper          from '@kitware/vtk.js/Rendering/Core/Mapper';
import vtkConeSource      from '@kitware/vtk.js/Filters/Sources/ConeSource';

export default {
  name: 'HelloWorld',

  setup() {
    const vtkContainer = ref(null);
    const context = ref(null);
    const coneResolution = ref(6);
    const representation = ref(2);

    function setConeResolution(res) {
      coneResolution.value = Number(res);
    }

    function setRepresentation(rep) {
      representation.value = Number(rep);
    }

    watchEffect(() => {
      const res = unref(coneResolution);
      const rep = unref(representation);
      if (context.value) {
        const { actor, coneSource, renderWindow } = context.value;
        coneSource.setResolution(res);
        actor.getProperty().setRepresentation(rep);
        renderWindow.render();
      }
    });

    onMounted(() => {
      if (!context.value) {
        const fullScreenRenderer = vtkFullScreenRenderWindow.newInstance({
          rootContainer: vtkContainer.value,
        });
        const coneSource = vtkConeSource.newInstance({ height: 1.0 });

        const mapper = vtkMapper.newInstance();
        mapper.setInputConnection(coneSource.getOutputPort());

        const actor = vtkActor.newInstance();
        actor.setMapper(mapper);

        const renderer = fullScreenRenderer.getRenderer();
        const renderWindow = fullScreenRenderer.getRenderWindow();

        renderer.addActor(actor);
        renderer.resetCamera();
        renderWindow.render();

        context.value = {
          fullScreenRenderer,
          renderWindow,
          renderer,
          coneSource,
          actor,
          mapper,
        };
      }
    });

    onBeforeUnmount(() => {
      if (context.value) {
        const { fullScreenRenderer, coneSource, actor, mapper } = context.value;
        actor.delete();
        mapper.delete();
        coneSource.delete();
        fullScreenRenderer.delete();
        context.value = null;
      }
    });

    return {
      vtkContainer,
      setRepresentation,
      setConeResolution,
      coneResolution,
      representation,
    };
  }
}
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

.app-container {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.title {
  position: absolute;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 10;
  color: white;
  font-size: 24px;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
  padding: 10px;
}

.main-layout {
  display: flex;
  flex: 1;
  height: 100%;
  overflow: hidden;
}

.vtk-canvas {
  flex: 1;
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
  z-index: 1;
}

.info-panel {
  width: 320px;
  background: rgba(255, 255, 255, 0.98);
  padding: 20px;
  overflow-y: auto;
  border-left: 1px solid #ddd;
  box-shadow: -2px 0 8px rgba(0, 0, 0, 0.1);
  font-family: Arial, sans-serif;
  line-height: 1.6;
  color: #333;
  position: relative;
  z-index: 2;
}

.info-panel h2 {
  font-size: 18px;
  margin-bottom: 12px;
  color: #222;
  border-bottom: 2px solid #4CAF50;
  padding-bottom: 8px;
}

.info-panel h3 {
  font-size: 14px;
  margin-top: 15px;
  margin-bottom: 8px;
  color: #444;
}

.info-panel p {
  font-size: 14px;
  margin-bottom: 15px;
  color: #555;
}

.info-panel ul {
  padding-left: 20px;
  margin-bottom: 15px;
}

.info-panel li {
  font-size: 13px;
  margin-bottom: 6px;
  color: #666;
}

.controls {
  position: fixed;
  top: 60px;
  left: 20px;
  background: rgba(255, 255, 255, 0.95);
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 100;
  border-collapse: collapse;
}

.controls td {
  padding: 8px;
  text-align: left;
}

.controls label {
  font-weight: bold;
  color: #333;
  margin-right: 10px;
}

.controls select,
.controls input {
  padding: 6px 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  min-width: 150px;
}

.controls select:hover,
.controls input:hover {
  border-color: #999;
}

.controls select:focus,
.controls input:focus {
  outline: none;
  border-color: #4CAF50;
  box-shadow: 0 0 4px rgba(76, 175, 80, 0.3);
}

@media (max-width: 768px) {
  .info-panel {
    display: none;
  }
}
</style>