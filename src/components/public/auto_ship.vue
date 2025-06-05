<template>
    <div id="cesiumContainer"></div>
  </template>
  
  <script>
  import 'cesium/Build/Cesium/Widgets/widgets.css'
  import * as Cesium from 'cesium'
  export default {
    mounted() {
      this.init()
    },
    methods: {
      async init() {
        Cesium.Ion.defaultAccessToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI3Nzc5YmFkNS0yMjgwLTQ3ZDAtYTI5My1mNmQ4MDQxM2YzYTkiLCJpZCI6MjA1NTgwLCJpYXQiOjE3MjM1MjAzOTd9.2WuAqjHxpHHSx0BYCsuWTpsvTwR67hfi8jTvmkzpwXM';

        const viewer = new Cesium.Viewer('cesiumContainer', {
          terrainProvider: await Cesium.createWorldTerrainAsync(),
          animation: false,
          timeline: false,
          selectionIndicator: true,
        })
        if (Cesium.FeatureDetection.supportsImageRenderingPixelated()) {
          //判断是否支持图像渲染像素化处理
          viewer.resolutionScale = window.devicePixelRatio
        }
        viewer.scene.postProcessStages.fxaa.enabled = true
        viewer.scene.debugShowFramesPerSecond = true // 显示帧率
  
        // Fly the camera to Denver, Colorado at the given longitude, latitude, and height.
        // viewer.camera.flyTo({
        // destination: Cesium.Cartesian3.fromDegrees(115.0565, 22.74248, 3000)
        // });

        // Add the 3D Tileset you created from your Cesium ion account.
        const model = await Cesium.Cesium3DTileset.fromIonAssetId(2994391);
        viewer.scene.primitives.add(model);

        viewer.zoomTo(model);

        
      },
    },
  }
  </script>
  
  <style lang="scss" scoped>
  #cesiumContainer {
    position: absolute;
    height: 100%;
    width: 100%;
  }
  </style>
  