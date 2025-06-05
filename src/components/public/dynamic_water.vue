<template>
  <div class="container">
    <div id="cesiumContainer"></div>
    <div class="control-panel">
      <div class="slider-container">
        <label>水面高度: {{ waterHeight }}米</label>
        <input type="range" v-model="waterHeight" :min="minHeight" :max="maxHeight" @input="updateWaterHeight">
      </div>
      <div class="button-container">
        <button @click="startDrawing" :class="{ active: isDrawing }">
          {{ isDrawing ? '完成绘制' : '绘制水面范围' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import 'cesium/Build/Cesium/Widgets/widgets.css'
import * as Cesium from 'cesium'

export default {
  data() {
    return {
      viewer: null,
      waterPrimitive: null,
      waterHeight: 730,
      minHeight: 200,
      maxHeight: 1000,
      drawingHandler: null,
      positions: [],
      polygon: null,
      isDrawing: false,
      hasPolygon: false,
      customPolygonCoordinates: null,  // 存储自定义多边形的坐标
      defaultRectangle: {
        west: 103.30,
        south: 29.25,
        east: 103.45,
        north: 29.35
      },
      rectangle: null,  // 添加这一行用于存储临时矩形
    }
  },
  mounted() {
    this.init()
  },
  methods: {
    async init() {
      this.viewer = new Cesium.Viewer('cesiumContainer', { 
        terrainProvider: await Cesium.createWorldTerrainAsync(),
        timeline: false,
        animation: false
      })
      if (Cesium.FeatureDetection.supportsImageRenderingPixelated()) {
        this.viewer.resolutionScale = window.devicePixelRatio
      }
      this.viewer.scene.postProcessStages.fxaa.enabled = true
      this.viewer.scene.globe.depthTestAgainstTerrain = true

      this.createWaterPrimitive()
      
      const center = Cesium.Cartesian3.fromDegrees(103.37, 29.15);
      this.viewer.camera.lookAt(center, new Cesium.Cartesian3(0.0, -47900.0, 39300.0));
      this.viewer.camera.lookAtTransform(Cesium.Matrix4.IDENTITY);
    },
    startDrawing() {
      if (this.isDrawing) {
        this.completeDrawing()
        return
      }

      this.isDrawing = true
      this.positions = []
      
      this.drawingHandler = new Cesium.ScreenSpaceEventHandler(this.viewer.scene.canvas)
      
      // 左键点击添加第一个点
      this.drawingHandler.setInputAction((event) => {
        const earthPosition = this.viewer.scene.pickPosition(event.position)
        if (Cesium.defined(earthPosition)) {
          if (this.positions.length === 0) {
            this.positions.push(earthPosition)
            this.createPolygonPreview()
          } else {
            this.completeDrawing()
          }
        }
      }, Cesium.ScreenSpaceEventType.LEFT_CLICK)
      
      // 鼠标移动时更新矩形
      this.drawingHandler.setInputAction((event) => {
        if (this.positions.length === 1) {
          const newPosition = this.viewer.scene.pickPosition(event.endPosition)
          if (Cesium.defined(newPosition)) {
            // 计算矩形的四个角点
            const startCartographic = Cesium.Cartographic.fromCartesian(this.positions[0])
            const endCartographic = Cesium.Cartographic.fromCartesian(newPosition)
            
            const west = Math.min(startCartographic.longitude, endCartographic.longitude)
            const east = Math.max(startCartographic.longitude, endCartographic.longitude)
            const south = Math.min(startCartographic.latitude, endCartographic.latitude)
            const north = Math.max(startCartographic.latitude, endCartographic.latitude)
            
            // 更新预览
            this.rectangle = {
              west: Cesium.Math.toDegrees(west),
              east: Cesium.Math.toDegrees(east),
              south: Cesium.Math.toDegrees(south),
              north: Cesium.Math.toDegrees(north)
            }
          }
        }
      }, Cesium.ScreenSpaceEventType.MOUSE_MOVE)
    },

    createPolygonPreview() {
      this.polygon = this.viewer.entities.add({
        rectangle: {
          coordinates: new Cesium.CallbackProperty(() => {
            if (!this.rectangle) return null
            return Cesium.Rectangle.fromDegrees(
              this.rectangle.west,
              this.rectangle.south,
              this.rectangle.east,
              this.rectangle.north
            )
          }, false),
          material: Cesium.Color.WHITE,
          height: this.waterHeight,
          outline: true,
          outlineColor: Cesium.Color.WHITE
        }
      })
    },

    completeDrawing() {
      if (this.drawingHandler) {
        this.drawingHandler.destroy()
        this.drawingHandler = null
      }
      
      if (this.rectangle) {
        if (this.polygon) {
          this.viewer.entities.remove(this.polygon)
        }
        
        this.customPolygonCoordinates = this.rectangle
        this.createWaterPrimitive(this.customPolygonCoordinates)
        this.hasPolygon = true
      }
      
      this.isDrawing = false
      this.positions = []
      this.rectangle = null
    },

    createWaterPrimitive(coordinates = null) {
      if (this.waterPrimitive) {
        this.viewer.scene.primitives.remove(this.waterPrimitive)
      }
      
      let geometry;
      if (coordinates && coordinates.west !== undefined) {
        geometry = new Cesium.RectangleGeometry({
          rectangle: Cesium.Rectangle.fromDegrees(
            coordinates.west,
            coordinates.south,
            coordinates.east,
            coordinates.north
          ),
          height: this.waterHeight,
          vertexFormat: Cesium.VertexFormat.POSITION_NORMAL_AND_ST,
        });
      } else {
        geometry = new Cesium.RectangleGeometry({
          rectangle: Cesium.Rectangle.fromDegrees(
            this.defaultRectangle.west,
            this.defaultRectangle.south,
            this.defaultRectangle.east,
            this.defaultRectangle.north
          ),
          height: this.waterHeight,
          vertexFormat: Cesium.VertexFormat.POSITION_NORMAL_AND_ST,
        });
      }

      // 创建水面材质
      const waterMaterial = new Cesium.Material({
        fabric: {
          type: "Water",
          uniforms: {
            baseWaterColor: new Cesium.Color(64 / 255.0, 157 / 255.0, 200 / 255.0, 0.5),
            normalMap: Cesium.buildModuleUrl("Assets/Textures/waterNormals.jpg"),
            frequency: 500.0,
            animationSpeed: 0.1,
            amplitude: 10,
            specularIntensity: 10
          }
        }
      });
      
      this.waterPrimitive = new Cesium.Primitive({
        geometryInstances: new Cesium.GeometryInstance({
          geometry: geometry
        }),
        appearance: new Cesium.EllipsoidSurfaceAppearance({
          material: waterMaterial,
          translucent: true
        }),
        show: true
      });
      
      this.viewer.scene.primitives.add(this.waterPrimitive);
    },
    updateWaterHeight() {
      this.createWaterPrimitive(this.customPolygonCoordinates)
    }
  },
  beforeUnmount() {
    if (this.drawingHandler) {
      this.drawingHandler.destroy()
    }
    if (this.viewer) {
      this.viewer.destroy()
    }
  }
}
</script>

<style lang="scss" scoped>
.container {
  position: relative;
  height: 100%;
  width: 100%;
}

#cesiumContainer {
  position: absolute;
  height: 100%;
  width: 100%;
}

.control-panel {
  position: absolute;
  top: 20px;
  right: 20px;
  background: rgba(255, 255, 255, 0.8);
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  
  .slider-container, .button-container {
    margin-bottom: 10px;
  }
}

.slider-container {
  display: flex;
  flex-direction: column;
  gap: 8px;
  
  label {
    font-size: 14px;
    color: #333;
  }
  
  input[type="range"] {
    width: 200px;
  }
}

.button-container {
  display: flex;
  gap: 8px;
  
  button {
    padding: 8px 12px;
    border: none;
    border-radius: 4px;
    background-color: #2196F3;
    color: white;
    cursor: pointer;
    font-size: 14px;
    transition: background-color 0.3s;
    
    &:hover {
      background-color: #1976D2;
    }
    
    &.active {
      background-color: #F44336;
      
      &:hover {
        background-color: #D32F2F;
      }
    }
  }
}
</style>
