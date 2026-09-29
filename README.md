# 天地图 UTS 原生插件（beige-tdt-map）

uni-app x（蒸汽模式）UTS 原生插件，GPU 渲染的高性能天地图地图组件，**Android / iOS / 鸿蒙 / Web 四端统一**。

| 端 | 渲染引擎 |
|---|---|
| Android / iOS | MapLibre Native（GPU OpenGL 渲染） |
| 鸿蒙 | 华为 MapKit |
| Web | MapLibre GL JS |

## ✨ 功能特性

- 🗺️ 三种地图类型：矢量 / 影像 / 地形
- 📍 标记点、批量标记、气泡（随地图移动跟随）、文字标注
- 🔷 折线 / 圆 / 矩形 / 多边形覆盖物
- 🖼️ 图片覆盖物（v1.9.0）：地理范围拉伸贴合、旋转/透明/层序/点击，http(s) 与本机缓存图
- 🔍 天地图地名搜索（v2 API）、内置搜索框与结果列表
- 🎯 定位（GPS / 网络双通道）、用户位置蓝点
- 🎛️ 缩放按钮 / 比例尺 / 版权条（v1.6.0）
- 📸 截图导出、坐标转换、手势控制
- 🎪 点击 / 长按 / 相机变化等事件
- ⚙️ 自定义瓦片、Android 纹理渲染模式

## 📦 快速开始

**1. 安装**：将 `uni_modules/beige-tdt-map/` 复制到你的 uni-app x 项目根目录下。

**2. 配置 Key**：编辑插件内 `config.uts`，填入你的天地图 Key（[申请地址](https://console.tianditu.gov.cn/)）：

```ts
const config: TDTMapGlobalConfig = {
    apiKey: '你的天地图服务端Key',   // App 端（Android/iOS/鸿蒙）
    webApiKey: '你的天地图浏览器端Key' // Web 端（需配置 Referer 白名单）
}
```

**3. 使用组件**：

```vue
<template>
  <view class="page">
    <tdt-map
      ref="mapRef"
      class="map"
      :latitude="39.9042"
      :longitude="116.4075"
      :zoom="12"
      map-type="vec"
      @mapReady="onMapReady"
      @mapClick="onMapClick"
    />
  </view>
</template>

<script setup lang="uts">
import { ref } from 'vue'

const mapRef = ref(null as any | null)

function onMapReady() {
  mapRef.value?.placeMarker(39.9142, 116.4074, '天安门')
}

function onMapClick(detail: MapClickDetail) {
  console.log('地图点击:', detail.lat, detail.lng)
}
</script>
```

## 📚 目录结构

| 目录 | 说明 |
|---|---|
| `uni_modules/beige-tdt-map/` | 插件本体（四端 UTS 实现 + uvue 组件） |
| `example-project/` | 示例工程（可直接导入 HBuilderX 运行） |
| `tmp-publish/` | 发布包（含 DCloud 市场发布配置与 changelog） |
| `docs/` | 设计文档与开发计划 |

## 📖 完整文档

组件全部 props、事件与 API 方法（标记、覆盖物、定位、信息窗口、截图等）见 [uni_modules/beige-tdt-map/README.md](uni_modules/beige-tdt-map/README.md)。

## 🔖 当前版本

**v1.9.0**（2026-09）：新增图片覆盖物（ImageOverlay，四端统一）。历史版本见 [changelog](tmp-publish/beige-tdt-map/changelog.md)。

## 📄 开源协议

[MIT License](LICENSE) —— 可自由使用、修改与商用，仅需保留版权声明。

## ⚠️ 合规说明

天地图服务条款要求在地图显著位置标注「天地图」标识与审图号。插件内置版权条默认显示「© 天地图」，上线前请通过 `copyrightText` prop 填入你的审图号（随 Key 申请获得）。
