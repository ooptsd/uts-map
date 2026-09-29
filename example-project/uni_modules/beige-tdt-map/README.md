# 天地图 UTS 原生插件（uni-app x 蒸汽模式）

基于 MapLibre（Android/iOS/Web）与华为 MapKit TileOverlay（鸿蒙）的天地图原生地图插件，GPU 渲染，四端统一 API。

**插件市场 ID**：`beige-tdt-map`

---

## 功能概览

| 模块 | 功能 |
|------|------|
| 🗺️ 底图 | 矢量 / 影像 / 地形三图切换，8 子域名并行加载 |
| 📍 标记 | 单标记、批量标记、文字标注，支持点击事件 `@markertap`，信息窗口气泡 |
| 📐 覆盖物 | 折线 / 圆 / 矩形 / 多边形，各平台原生渲染 |
| 🎯 覆盖物管理 | ID 追踪系统，支持单个移除、显隐控制 |
| 🧭 坐标转换 | GCJ02 → WGS84（三端纯算法，厘米级精度） |
| 🔍 视野控制 | 放大/缩小/自适应/像素平移/多点视野/布局刷新 |
| ✋ 手势 | 缩放/拖拽开关，旋转/倾斜手势控制，最大/最小缩放限制 |
| 🔄 旋转倾斜 | bearing 旋转（0-360°）/ tilt 倾斜（0-60°），手势与程序双控 |
| 🖐 交互增强 | 地图长按事件 `@mapLongClick`、相机移动完成事件 `@cameraChange` |
| 📍 定位 | 显示/隐藏用户位置蓝点，获取当前 GPS 坐标（三端原生定位） |
| 🖼️ 自定义瓦片 | 叠加外部 XYZ 瓦片图层（如天气、交通、专题图） |
| 🧮 信息计算 | Haversine 测距、最佳视野计算、坐标系查询、容器尺寸 |
| 🌐 天地图 API | 地理编码 / 逆地理编码 / 关键字搜索 / 范围搜索 / 周边搜索 |
| 📸 截图 | 地图快照导出 PNG（iOS `drawHierarchy` / Android snapshot） |
| 📡 生命周期 | `onResume` / `onPause` / `destroyMap` |
| 🧩 内置 UI | 搜索框 / 搜索结果列表 / 定位按钮 / marker 气泡（随地图移动跟随） |
| 🎛️ 内置控件 | 缩放按钮、比例尺、版权条（**v1.6.0 新增**） |
| 📊 数据可视化 | 标记聚合（聚合簇带数量徽标，点击自动放大）、热力图（权重/半径/透明度）（**v1.7.0 新增**） |

---

## 快速开始

### 安装

1. 在 uni-app x 项目根目录创建 `uni_modules/beige-tdt-map/`
2. 将本插件所有文件放入该目录
3. 配置 API Key（见下方）

### 配置 API Key

在 `uni_modules/beige-tdt-map/config.uts` 中填写你的天地图 Key（双字段：`apiKey` 供 App 端使用，`webApiKey` 供 Web 端使用）：

```ts
type TDTMapGlobalConfig = {
    apiKey: string
    webApiKey: string
}

const config: TDTMapGlobalConfig = {
    apiKey: '你的天地图服务端Key',
    webApiKey: '你的天地图浏览器端Key'
}

export default config
```

| 字段 | 适用端 | Key 类型要求 |
|------|--------|-------------|
| `apiKey` | App 端（Android/iOS/鸿蒙） | 「**服务端**」或「**Android 平台**」类型 |
| `webApiKey` | Web 端（浏览器） | 「**浏览器端**」类型，需在控制台设置 Referer 白名单 |

> ⚠️ **Key 类型必须匹配**：原生 App 必须使用「**服务端**」或「**Android 平台**」类型 Key，**不能使用浏览器端 Key**（浏览器端 Key 校验 Referer 头，原生 App 不发送 Referer 导致 403）。
>
> Web 端同理：只配置 `apiKey` 而未配置 `webApiKey` 时，Web 端会 fallback 使用 `apiKey`（可能同样出现 403），建议两字段分别配置。
>
> 组件 props 传入 `apiKey` 时优先级高于 config.uts（适合多 Key 场景）。
>
> 申请地址：https://console.tianditu.gov.cn

### 基础用法

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
      @markertap="onMarkerTap"
    />
  </view>
</template>

<script setup lang="uts">
import { ref } from 'vue'
import type { MapClickDetail } from '@/uni_modules/beige-tdt-map'

const mapRef = ref(null as any | null)
const isReady = ref(false)

function onMapReady() {
  isReady.value = true
  // 地图就绪后可调用 API
  mapRef.value?.placeMarker(39.9142, 116.4074, '天安门')
}

function onMapClick(detail: MapClickDetail) {
  console.log('地图点击:', detail.lat, detail.lng)
}

function onMarkerTap(detail: MapClickDetail) {
  console.log('标记点击:', detail.lat, detail.lng)
}
</script>

<style>
.page { flex: 1; }
.map { flex: 1; width: 100%; }
</style>
```

---

## 组件属性

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `apiKey` | String | `''` | 天地图 Key（优先级高于 config.uts） |
| `latitude` | Number | `39.9042` | 初始纬度 |
| `longitude` | Number | `116.4075` | 初始经度 |
| `zoom` | Number | `10` | 初始缩放级别（1-18） |
| `mapType` | String | `'vec'` | 地图类型：`vec` 矢量 / `img` 影像 / `ter` 地形 |
| `showSearchBar` | Boolean | `true` | 内置搜索框开关（**v1.5.0 新增**） |
| `showLocateButton` | Boolean | `true` | 内置定位按钮开关（**v1.5.0 新增**） |
| `textureMode` | Boolean | `false` | Android 纹理渲染模式，模拟器/特殊层叠场景可开启（**v1.5.1 新增**） |
| `showZoomControl` | Boolean | `true` | 缩放按钮显隐（Android/iOS 为 uvue 样式；鸿蒙/Web 为原生样式，仅初始化生效）（**v1.6.0 新增**） |
| `showScaleControl` | Boolean | `true` | 比例尺显隐（同上，鸿蒙/Web 仅初始化生效）（**v1.6.0 新增**） |
| `showCopyrightControl` | Boolean | `true` | 版权条显隐（四端统一 uvue 层，天地图合规要求，默认开启；响应式）（**v1.6.0 新增**） |
| `copyrightText` | String | `'© 天地图'` | 版权条文本。合规提示：建议填入你的天地图审图号，如"© 天地图 GS(2025)xxx号"（审图号随 Key 申请获得，需用户填写）（**v1.6.0 新增**） |

## 组件事件

| 事件 | 参数 | 说明 |
|------|------|------|
| `@mapReady` | 无 | 地图初始化完成 |
| `@mapClick` | `{ lat, lng }` | 地图空白区域点击 |
| `@markertap` | `{ lat, lng }` | 标记点点击（**v1.2.0 新增**） |
| `@mapLongClick` | `{ lat, lng }` | 地图长按（**v1.4.0 新增**） |
| `@cameraChange` | `{ lat, lng }` | 相机视野变化完成（**v1.4.0 新增**） |
| `@mapError` | `{ code, message }` | 错误回调 |

---

## API 参考

通过 `ref` 调用组件暴露的方法：

```ts
const mapRef = ref(null as any | null)
mapRef.value?.方法名(...)
```

### 地图控制

| 方法 | 参数 | 说明 |
|------|------|------|
| `moveToCenter(lat, lng)` | 坐标 | 移动地图中心 |
| `zoomTo(level)` | 缩放级别 | 设置缩放级别 |
| `getCurrentCenter()` | 无 | 获取当前中心坐标 |
| `getCurrentZoom()` | 无 | 获取当前缩放级别 |
| `changeMapType(type)` | `'vec'/'img'/'ter'` | 切换地图类型 |
| `fitToBounds(minLat, minLng, maxLat, maxLng, padding)` | 边界+边距 | 自适应视野 |
| `zoomInOne()` | 无 | 放大一级 |
| `zoomOutOne()` | 无 | 缩小一级 |
| `setMinZoomLevel(zoom)` | 数值 | 最小缩放限制 |
| `setMaxZoomLevel(zoom)` | 数值 | 最大缩放限制 |
| `panToLocation(lat, lng, zoom?)` | 坐标+缩放 | 平移到指定位置 |
| `centerAtZoom(lat, lng, zoom)` | 坐标+缩放 | 居中并缩放 |
| `setZoomEnabled(bool)` | 布尔 | 缩放手势开关 |
| `setDragEnabled(bool)` | 布尔 | 拖拽手势开关 |
| `setMapMaxBounds(minLat, minLng, maxLat, maxLng)` | 边界 | 设置视野边界（v1.8.0: 任一参数传 null 清除限制） |
| **🆕** `doCheckResize()` | 无 | 触发布局刷新 |
| **🆕** `doPlanBy(x, y)` | 像素偏移 | 像素级平移 |
| **🆕** `setViewportPoints(points)` | 坐标数组 | 多点自适应视野 |

### 标记点

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `placeMarker(lat, lng, title, snippet?)` | `number` (覆盖物ID) | 添加单个标记（**v1.3.0 新增 snippet 参数**） |
| `batchMarkers([...])` | `Array<number>` | 批量添加标记 |
| `clearAllMarkers()` | 无 | 清除所有标记 |
| **🆕** `drawLabel(lat, lng, text)` | `number` | 添加文字标注 |

### 📍 定位（v1.3.0 新增）

| 方法 | 说明 |
|------|------|
| `setShowUserLocation(enabled)` | 显示/隐藏用户位置蓝点 |
| `getCurrentLocation(callback, onError?)` | 获取当前 GPS 坐标，异步回调 |

### 💬 信息窗口（v1.3.0 新增）

| 方法 | 说明 |
|------|------|
| `showMarkerInfoById(id)` | 显示指定标记的信息窗口 |
| `hideMarkerInfoById(id)` | 隐藏指定标记的信息窗口 |

### 💬 marker 气泡（v1.5.6 新增）

uvue 层自定义气泡（MapLibre 11.x 原生 InfoWindow 在 native-view 集成下不显示，改 Web Mercator 数学换算定位），随地图移动/缩放自动跟随，四端统一 css px。

| 方法 | 参数 | 说明 |
|------|------|------|
| `showBubble(text, lat, lng)` | 文本 + 坐标 | 显示自定义气泡（点击气泡关闭） |
| `hideBubble()` | 无 | 隐藏气泡 |

### 🖼️ 自定义瓦片（v1.3.0 新增）

| 方法 | 说明 |
|------|------|
| `addTileLayer(id, tileUrl, options?)` | 添加 XYZ 瓦片图层（v1.8.0: options 为 `{ opacity?, minZoom?, maxZoom?, zIndex? }`，zIndex 仅支持 0 与缺省） |
| `removeTileLayer(id)` | 移除指定瓦片图层 |

### 🧩 内置 UI 工具（v1.5.0 / v1.5.6）

| 方法 | 说明 |
|------|------|
| `doSearch(keyword?)` | 执行关键字搜索（使用内置搜索框，v1.5.0） |
| `selectSearchResult(index)` | 选中搜索结果并定位到该点（v1.5.0） |
| `locateToCurrent()` | 定位到当前位置并回中（v1.5.0） |
| `isMapReady()` | 返回地图是否就绪（v1.5.0） |

### 🔄 旋转与倾斜（v1.4.0 新增）

| 方法 | 参数 | 说明 |
|------|------|------|
| `setMapBearing(bearing)` | 0-360° | 设置地图旋转角度（0°=正北，顺时针） |
| `setMapTilt(tilt)` | 0-60° | 设置地图倾斜角度（0°=垂直俯瞰） |
| `getMapBearing()` | 无 | 获取当前旋转角度 |
| `getMapTilt()` | 无 | 获取当前倾斜角度 |

```ts
// 向东旋转 90°
mapRef.value?.setMapBearing(90)

// 倾斜 45° 俯瞰
mapRef.value?.setMapTilt(45)

// 回正
mapRef.value?.setMapBearing(0)
mapRef.value?.setMapTilt(0)
```

### ✋ 手势进阶（v1.4.0 新增）

| 方法 | 参数 | 说明 |
|------|------|------|
| `setRotateEnabled(enabled)` | 布尔 | 双指旋转手势开关 |
| `setTiltEnabled(enabled)` | 布尔 | 双指倾斜手势开关（鸿蒙 4.1.0+ 支持） |

```ts
// 禁止用户旋转地图
mapRef.value?.setRotateEnabled(false)
```

### 📊 数据可视化（v1.7.0 新增）

| 方法 | 参数 | 说明 |
|------|------|------|
| `addMarkerCluster(points, options?)` | 点数组 + 配置 | 标记聚合（聚合簇带数量徽标，点击自动放大展开；重复调用自动重建；options 为 `{ clusterRadius?, clusterMaxZoom? }`） |
| `removeMarkerCluster()` | 无 | 移除标记聚合图层 |
| `addHeatmap(points, options?)` | 点数组 + 配置 | 热力图（密度可视化；options 为 `{ radius?, opacity? }`；鸿蒙需 HarmonyOS 6.0+，低版本显式报错） |
| `removeHeatmap()` | 无 | 移除热力图图层 |

```ts
// 聚合：points 为 { lat, lng } 数组
mapRef.value?.addMarkerCluster(points, { clusterRadius: 50, clusterMaxZoom: 14 })
mapRef.value?.removeMarkerCluster()

// 热力图：points 为 { lat, lng, weight? } 数组
mapRef.value?.addHeatmap(heatPoints, { radius: 30, opacity: 0.6 })
mapRef.value?.removeHeatmap()
```

### 🎨 图层增强（v1.8.0 新增）

| 方法 | 参数 | 说明 |
|------|------|------|
| `setStyle(styleJson)` | MapLibre Style Spec JSON | 运行时替换地图样式（须含 `version` 字段，否则报 code=4 不切换）。切换后自动重建此前添加的用户图层（自定义瓦片/WMS/聚合/热力）；天底图由自定义样式自带，插件不自动加回。鸿蒙上报 code=5 |
| `removeStyle()` | 无 | 还原天地图默认底图并重建用户图层；未调用过 setStyle 时调用不产生可见变化（幂等） |
| `addWMSTileLayer(id, wmsUrl, layers, options?)` | WMS 服务参数 | 叠加 WMS 图层（自动拼装 GetMap 请求；移除仍用 `removeTileLayer(id)`）。Android/iOS/Web 支持；鸿蒙上报 code=5 |
| `isSupportCanvas()` | 无 | 当前平台是否支持 canvas 标注（Web=true；Android/iOS/鸿蒙=false） |

```ts
// 切换为自定义样式（OSM 底图示例）
mapRef.value?.setStyle(JSON.stringify({
  version: 8,
  glyphs: 'https://demotiles.maplibre.org/font/{fontstack}/{range}.pbf',
  sources: { osm: { type: 'raster', tiles: ['https://tile.openstreetmap.org/{z}/{x}/{y}.png'], tileSize: 256 } },
  layers: [{ id: 'osm-layer', type: 'raster', source: 'osm' }]
}))

// 还原天地图底图（用户图层自动重建）
mapRef.value?.removeStyle()

// WMS 叠加（贴底 + 80% 不透明度；Terrestris 公共 OSM WMS，实测可用）
mapRef.value?.addWMSTileLayer('wms-demo', 'https://ows.terrestris.de/osm/service', 'OSM-WMS', { opacity: 0.8, zIndex: 0 })

// 清除边界限制
mapRef.value?.setMapMaxBounds(null, null, null, null)
```

> ⚠️ 平台差异：鸿蒙 MapKit 的样式 schema 与 MapLibre style spec 不兼容，`setStyle`/`addWMSTileLayer` 会通过 `mapError` 事件上报 code=5（平台不支持），`isSupportCanvas` 返回 false。
> ⚠️ 鸿蒙 `addTileLayer` 的 options 仅 opacity（transparency）生效，minZoom/maxZoom/zIndex 忽略并 warn。
> `zIndex` 仅支持 `0`（插到底图注记层之下）与缺省（默认置顶），其他值忽略并 warn。

### 🖼️ 图片覆盖物（v1.9.0 新增）

| 方法 | 参数 | 说明 |
|------|------|------|
| `addImageOverlay(params)` | `ImageOverlayOptions` | 添加图片覆盖物：图片拉伸填满 bounds（bearing 旋转），返回 overlay ID；与 `removeOverlayById`/`showOverlayById`/`hideOverlayById`/`clearAllOverlays` 统一管理 |
| `updateImageOverlay(id, changes)` | overlay ID + `ImageOverlayUpdateOptions` | 运行时更新（全可选字段，未传项保持不变；ID 不变） |
| `@imageOverlayClick` | 事件 | `clickable=true` 且可见时触发，互斥消费 mapClick（四端行为一致；detail 含 id/lat/lng） |

```ts
const id = mapRef.value?.addImageOverlay({
  southwest: { lat: 39.88, lng: 116.36 },
  northeast: { lat: 39.94, lng: 116.44 },
  image: 'https://example.com/radar.gif',   // http(s) URL 或本机缓存绝对路径（缓存由调用方维护，插件仅按地址加载）
  transparency: 0.3,   // 0=不透明（默认）~ 1=全透明
  zIndex: 1,           // 0=注记层下 / 缺省=置顶 / 正整数=图片层之间按值排序
  bearing: 0,          // 0~360 顺时针，绕 bounds 中心
  visible: true,
  clickable: true
})

// 半透明 + 旋转 45°（ID 不变）
mapRef.value?.updateImageOverlay(id, { transparency: 0.5, bearing: 45 })

// 移除/显隐复用统一管理体系
mapRef.value?.removeOverlayById(id)
```

> ⚠️ 平台差异：
> - `image` 四端均支持 http(s) URL 与本机缓存文件绝对路径（缓存由调用方维护）；**Web 端仅支持 http(s)/data: URL**（浏览器无法读本机路径），传本机路径上报 code=4 并返回 0
> - 大图建议最长边 ≤ 4096px：Android/鸿蒙超大图自动下采样（inSampleSize/desiredSize），iOS/Web 由引擎处理，超限可能渲染失败
> - `zIndex` 为三档语义（对齐瓦片图层约定），仅决定图片覆盖物之间的相对层序，不改变与 marker 的相对关系；鸿蒙原生精确生效
> - `bearing` 在 MapLibre 三端通过四角坐标在 Web Mercator 平面旋转等效实现；鸿蒙原生参数直通
> - 鸿蒙特有 `anchorU/anchorV`（默认 0.5/0.5）未纳入跨端 API——MapLibre 图片恒铺满四角范围，无锚点语义
> - 图片异步加载：ID 立即可用于 remove/show/hide/update（加载中的 update 合并在加载完成后应用）；加载失败经 `mapError` 上报 code=2（网络）/code=4（解码或参数非法）且该 ID 失效
> - MapLibre 三端 `setStyle` 后图片覆盖物随用户图层自动重建（同 ID 保序）；鸿蒙无 setStyle

### 覆盖物

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `drawPolyline(points, color, width)` | `number` | 绘制折线 |
| `drawCircle(lat, lng, radius, color, fillColor)` | `number` | 绘制圆 |
| `drawRectangle(minLat, minLng, maxLat, maxLng, color, fillColor)` | `number` | 绘制矩形 |
| `drawPolygon(points, color, fillColor)` | `number` | 绘制多边形 |
| `clearAllOverlays()` | 无 | 清除所有覆盖物 |

### 🆕 覆盖物管理（v1.2.0）

| 方法 | 参数 | 说明 |
|------|------|------|
| `removeOverlayById(id)` | overlay ID | 按 ID 移除单个覆盖物 |
| `showOverlayById(id)` | overlay ID | 按 ID 显示覆盖物 |
| `hideOverlayById(id)` | overlay ID | 按 ID 隐藏覆盖物 |

```ts
// 示例：添加标记并追踪 ID
const markerId = mapRef.value?.placeMarker(39.9142, 116.4074, '北京')

// 隐藏它
mapRef.value?.hideOverlayById(markerId)

// 重新显示
mapRef.value?.showOverlayById(markerId)

// 永久移除
mapRef.value?.removeOverlayById(markerId)
```

### 坐标转换

| 方法 | 参数 | 说明 |
|------|------|------|
| `convertGcj02ToWgs84(lat, lng)` | GCJ02 坐标 | 返回 WGS84 坐标 |

### 截图 & 信息

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `takeSnapshot(callback)` | `void` | 异步截图，回调返回 PNG ArrayBuffer 数据 |
| `showCenter()` | 显示到 infoText | 当前中心坐标 |
| `showZoom()` | 显示到 infoText | 当前缩放级别 |
| `showBounds()` | 显示到 infoText | 当前视野范围 |
| `getCurrentBounds()` | `MapBoundsDetail` | 获取视野范围 |

### 🆕 信息计算（v1.2.0）

| 方法 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| `calculateDistance(start, end)` | 两点坐标 | `number`（米） | Haversine 测距 |
| `calculateViewport(points)` | 坐标数组 | `{ center, zoom }` | 计算最佳视野 |
| `getMapCode()` | 无 | `string` | 返回 `"EPSG:4326"` |
| `getMapSize()` | 无 | `{ width, height }` | 容器像素尺寸 |

### 🆕 天地图 Web API（v1.2.0）

| 方法 | 参数 | 说明 |
|------|------|------|
| `reverseGeocode(latlng, callback)` | 坐标 + 回调 | 逆地理编码（坐标→地址） |
| `forwardGeocode(address, callback)` | 地址 + 回调 | 地理编码（地址→坐标） |
| `keywordSearch(params, callback)` | 搜索参数 + 回调 | 关键字搜索 |
| `boundsSearch(params, callback)` | 搜索参数 + 回调 | 视野内搜索 |
| `nearbySearch(params, callback)` | 搜索参数 + 回调 | 周边搜索 |

```ts
// 逆地理编码示例
mapRef.value?.reverseGeocode(
  { lat: 39.9088, lng: 116.3975 },
  (data: UTSJSONObject) => {
    console.log('地址:', data.getString('formatted_address'))
  }
)

// 关键字搜索示例
mapRef.value?.keywordSearch(
  { keyWord: '酒店', start: '0', count: '10' },
  (data: UTSJSONObject) => {
    console.log('搜索结果:', JSON.stringify(data))
  }
)

// 周边搜索示例
mapRef.value?.nearbySearch(
  { keyWord: '餐厅', pointLonlat: '116.3975,39.9088', queryRadius: '1000' },
  (data: UTSJSONObject) => {
    console.log('周边:', JSON.stringify(data))
  }
)
```

---

## 平台兼容性

| 平台 | 支持 | 引擎 |
|------|------|------|
| Android | ✅ | MapLibre Android |
| iOS | ✅ | MapLibre iOS |
| HarmonyOS | ✅ | 华为 MapKit TileOverlay |
| H5 / Web | ✅ | MapLibre GL JS（CDN 动态加载） |
| 小程序 | ❌ | 不支持 |

> 💡 **Web 端说明**：Web 平台通过条件编译（`#ifdef WEB`）自动切换为 MapLibre GL JS 实现，`<native-view>` 替换为 `<view>` 容器，MapLibre GL JS 从 unpkg CDN 按需加载。全部 62 个 API 与 App 端保持一致。

---

## 类型定义

```ts
type MapClickDetail = { lat: number; lng: number }
type MapBoundsDetail = { minLat: number; minLng: number; maxLat: number; maxLng: number }
type MapErrorDetail = { code: number; message: string }
type MapSizeDetail = { width: number; height: number }
type MapViewportDetail = { center: MapClickDetail; zoom: number }
type MapClickCallback = (detail: MapClickDetail) => void
type MapReadyCallback = () => void
type MapErrorCallback = (detail: MapErrorDetail) => void
type MarkerClickCallback = (detail: MapClickDetail) => void
type MapLongClickCallback = (detail: MapClickDetail) => void
type CameraChangeCallback = (detail: MapClickDetail) => void
type GeocodeCallback = (data: UTSJSONObject) => void

type SearchOptions = {
  keyWord: string
  mapBound?: string
  level?: string
  queryType?: string
  queryRadius?: string
  pointLonlat?: string
  start?: string
  count?: string
  specifyAdminCode?: string
}

// v1.9.0：图片覆盖物
type ImageOverlayOptions = {
  southwest: MapClickDetail
  northeast: MapClickDetail
  image: string                // http(s) URL 或本机缓存绝对路径（Web 仅 http(s)/data:）
  transparency?: number        // 0=不透明（默认）~ 1=全透明
  zIndex?: number              // 0=注记层下 / 缺省=置顶 / 正整数=图片层之间按值排序
  bearing?: number             // 0~360 顺时针，绕 bounds 中心
  visible?: boolean            // 默认 true
  clickable?: boolean          // 默认 true（命中点击时触发 imageOverlayClick 并互斥消费 mapClick）
}
type ImageOverlayUpdateOptions = {
  southwest?: MapClickDetail
  northeast?: MapClickDetail
  image?: string
  transparency?: number
  zIndex?: number
  bearing?: number
  visible?: boolean
  clickable?: boolean
}
type ImageOverlayClickDetail = { id: number; lat: number; lng: number }
```

---

## 版本历史

### v1.9.0（2026-09）

#### ✨ 新增：图片覆盖物（ImageOverlay）

- 四端统一的图片地理范围覆盖：图片拉伸填满 bounds，bearing 绕中心旋转（MapLibre 三端 Web Mercator 四角等效实现，鸿蒙原生直通）
- 图片来源：http(s) URL 与本机缓存绝对路径（缓存由调用方维护）；Web 端仅 http(s)/data: URL（本机路径上报 code=4）
- 大图防护：Android inSampleSize / 鸿蒙 desiredSize 两段解码链制 4096px；iOS/Web 引擎直载
- 点击互斥：四端 clickable 命中消费 mapClick；MapLibre 三端几何 hit-test（raster 无要素），鸿蒙原生 imageOverlayClick
- 生命周期：异步加载 ID 立即可管理（加载中 update 合并）、失败上报 code=2/4 且 ID 失效、setStyle 自动重建（MapLibre 三端同 ID）

### v1.6.0（2026-08-16）

#### ✨ 新增：基础控件组（缩放按钮 / 比例尺 / 版权条）

- 缩放按钮：Android/iOS 为 uvue 层样式（调 `zoomInOne`/`zoomOutOne`，原生 SDK 无内置控件）；鸿蒙为原生 `zoomControlsEnabled`；Web 为原生 `NavigationControl`
- 比例尺：Android/iOS 为 uvue 层公式计算（OSM Wiki ground resolution，[1,2,5]×10ⁿ 档位取整）；鸿蒙为原生 `scaleControlsEnabled`；Web 为原生 `ScaleControl`
- 版权条：四端统一 uvue 层（天地图服务条款要求在显著位置标注"天地图"与审图号），默认"© 天地图"，可通过 `copyrightText` 填入审图号
- 新增 4 个 props：`showZoomControl` / `showScaleControl` / `showCopyrightControl` / `copyrightText`
- 动态切换限制：`showZoomControl`/`showScaleControl` 在鸿蒙/Web 原生端为初始化一次性配置（同 `textureMode` 限制）；Android/iOS 的 uvue 控件与 `copyrightText`/`showCopyrightControl` 天然响应式
- 隐藏 MapLibre 原生 logo 与 attribution（Android/iOS/Web），鸿蒙华为 logo 移位右下角（BOTTOM_END）——避免与 uvue 版权条重叠及错误版权信息
- 修复鸿蒙 logo 位置 TOP_END→BOTTOM_END：右上角与华为默认指南针（旋转/倾斜时出现）重叠；BOTTOM_END 为华为官方示例位置，与缩放控件共存
- 修复 Android 缩放按钮无响应：`zoomIn`/`zoomOut` 沿用 v1.5.5 实测结论改用 `moveCamera` 瞬时缩放（`animateCamera` 动画在 UTS 集成下不推进）

### v1.5.5 ⚡（2026-08）

#### ⚡ 优化：Android 双指缩放掉帧（中端机）

- 根因：MapLibre 默认 `pixelRatio` = 设备屏幕密度（1080p 中端机约 2.75~3.0），双指缩放时整个 GL 表面每帧重采样，中端 GPU 光栅化负载过重导致掉帧（context7 验证 MapLibre 官方源码 `MapLibreMapOptions.java`）
- 初始化时 `pixelRatio` 限制为 2.0（渲染像素量约减半，清晰度损失轻微；低密度屏保持原值不受影响）
- `setPrefetchZoomDelta(2)` 预加载相邻缩放级别瓦片，缩放时新层级瓦片更快就绪

#### 🐛 修复：animateCamera 动画不推进（跳转目标不可达）

- 日志实测：MapLibre 11.x `animateCamera` 动画在 UTS native-view 集成下不推进（500ms 零进展，目标 zoom 中间值从未出现）
- `panTo` / `centerAndZoom` / 选中搜索结果定位改用 `moveCamera` 瞬时跳转

#### 🐛 修复：搜索结果列表点击穿透到地图

- 根因：结果列表是搜索栏（`height:70rpx` + `overflow:visible`）的溢出子元素，Android 端溢出区域 hit-test 失效，点击列表项穿透到 native-view（日志实测触发的是地图点击）
- 修复：结果列表改为独立容器，绝对定位在搜索框正下方

---

### v1.5.6 🎉（2026-08）

#### 🆕 新增：marker 气泡（`showBubble` / `hideBubble`）

- 根因（日志实测）：MapLibre 11.x `showInfoWindow` 调用无异常但气泡不显示，与 animateCamera 动画不推进同源——SDK 内 View 动态更新机制在 native-view 集成下失效
- 方案：uvue 层自定义气泡 + Web Mercator 数学换算（worldSize = 256 × 2^zoom），零原生依赖、热刷生效
- 气泡随地图移动/缩放自动跟随（`cameraChange` 事件重算位置），点击气泡关闭

#### 🐛 修复：气泡坐标四端单位不一致

- 各端 UTS `getSize` 单位各异：Android 物理像素 / iOS point（逻辑像素）/ 鸿蒙恒返回 `{0,0}` / H5 css px，原公式按 Android 物理像素设计，iOS/鸿蒙/H5 必错位
- 改用 uni-app x 官方 API `uni.getElementById().getBoundingClientRect()`（同步返回 css px，兼容性：Web 4.0 / Android 4.0 / iOS 4.11 / HarmonyOS 4.61）自测容器尺寸，四端统一

#### 🐛 修复：跨运行时 UTSJSONObject 下标访问失败

- UTS 回调中跨运行时传递的 `UTSJSONObject` 方法/下标访问受限
- 修复：`JSON.parse` 本地重建后使用 `[]` 下标访问（uni-app x 官方 JSON 文档全平台通用方式）

#### 🐛 修复：天地图 v2 搜索未传 count 参数

- v2 搜索接口需显式传 `count`，否则返回条数异常；`search` / `searchInBounds` / `searchNearby` 已统一透传

---

### v1.5.7 🔬（2026-08）

#### 🔬 新增：Android 性能诊断日志（`TDTMap: [PERF]`）

- 手势帧统计：`addOnCameraMoveStartedListener` + `addOnCameraMoveListener` 统计逻辑帧，`cameraIdle` 时汇总输出一条日志（避免逐帧日志加剧卡顿）
- 渲染帧统计：`addOnDidFinishRenderingFrameListener` 统计真实画面帧 + GPU 编码/渲染耗时，与逻辑帧对比定位「逻辑快画面卡」
- 判读标准：平均帧间隔 ≈16.7ms=60fps / ≈33ms=30fps / >50ms 明显卡

---

### v1.5.8 🔬（2026-08）

#### 🔬 扩展：相机移动统计覆盖全部来源

- 统计范围从仅手势（reason=1）扩展到所有相机移动：reason 1=手势 / 2=开发者动画 / 3=API 动画（含 fling 惯性）
- 用于排查松手后惯性滑动是否触发与推进

---

### v1.5.4 🐛（2026-08）

#### 🐛 修复：搜索/地点搜索解析失败（天地图老搜索接口已下线）

**根因**：天地图老接口 `api.tianditu.gov.cn/search` 已下线（404 返回非 JSON），`JSON.parse` 对非 JSON 返回 null，`null as UTSJSONObject` 抛 "null cannot be cast" 崩溃；且 v2 响应结构变化（status 为对象、pois 在顶层）。

- 四端（Android/iOS/鸿蒙/Web）搜索统一迁移到官方「地名搜索2.0」`/v2/search`（queryType=12 行政区划搜索，specify 默认 156000000=中国）
- 四端 HTTP 请求解析全部增加 JSON.parse 判空防御（非 JSON 响应回调结构化错误，不再崩溃）
- 组件内置搜索适配 v2 响应格式（status 对象 infocode==1000 判定、顶层 pois）
- demo 页搜索展示适配 v2（infocode + 结果总数）

#### 🐛 修复：Android 定位按钮点击无反应

**根因**：Android 6+ 需运行时定位授权，原生 `LocationManager` **不会自动弹授权框**（仅 `uni.getLocation` 等框架 API 会自动弹框），无权限时静默失败；且失败信息仅 `console.error`，用户看不到任何反馈。

- Android 端 `getUserLocation` 新增**运行时权限检查与申请**（`UTSAndroid.requestSystemPermission` 官方 API），授权成功后自动继续定位
- `showUserLocation`（定位蓝点）同样先检查权限再激活（MapLibre `LocationComponent` 不会自动弹授权框）
- **定位服务开关检测**：GPS 与网络 provider 均关闭时立即报错，不再干等 10 秒超时
- **GPS 与网络双通道并发单次定位（先回先得）**：室内/无星历环境网络定位 1-3 秒内返回，修复仅请求 GPS 时冷启动数分钟无回调
- 组件内置定位按钮增加**即时反馈**：点击显示「定位中...」，失败 toast 显示具体原因（权限拒绝/定位服务未开启/超时）
- demo 页移除 `uni.authorize` 残留（uni-app x 无此 API，调用会直接报错）

#### ⚡ 优化：Android 滑动卡顿

- 底图/注记两个 raster 图层设置 `raster-fade-duration: 0`——MapLibre Style Spec 默认 300ms 淡入，快速滑动时新瓦片半透明过渡造成视觉卡糊，设为 0 瓦片即时显示

### v1.5.3 🐛（2026-08）

#### 🐛 Bug 修复（真机报错专项修复）

**定位按钮报错 `uni.authorize is not a function`**

- uni-app x **不存在** `uni.authorize` API（仅存在于 uni-app Vue2/Vue3），组件内置定位按钮误调用导致 App Error
- 修复：移除 `uni.authorize`，直接调用定位 API——uni-app x 首次调用定位会**自动弹出系统授权框**
- ⚠️ 推翻 v1.5.0 中「demo 页增加 uni.authorize 运行时权限请求」的修复方案

**UTS 回调报错 `回调函数已释放，不能再次执行`**

- HBuilderX 4.25+ 中 UTS 插件导出函数的回调参数默认**触发一次后自动回收**，二次触发即报错
- 修复：四端（Android/iOS/鸿蒙/Web）10 个带回调的导出函数全部添加 `@UTSJS.keepAlive` 装饰器（initMap、setMarkerClickCallback、setMapLongClickCallback、setCameraChangeCallback、geocode_getLocation、geocode_getPoint、search、searchInBounds、searchNearby、getUserLocation）

**搜索/地理编码报错 `HTTP 请求失败: null`**

- 原 Android 实现用 `HttpURLConnection` 在 UTS 调用线程（主线程）**同步请求**，触发 `NetworkOnMainThreadException`（message 为 null）
- 修复：改用 `uni.request` **异步**请求（框架自动调度网络线程执行）
- 修复：`res.data` 类型处理——`dataType` 默认 `json` 时已是 UTSJSONObject，原 `as string` 强转抛转换异常，改为 `typeof` 分支兼容 string 与 UTSJSONObject

**Web 端内联类型修复**

- Web 端残留 3 处内联对象字面量类型（违反 UTS 规范），统一为命名类型 `MarkerInfo` / `TileLayerInfo` / `TdtLayerConfig`

---

### v1.5.2 🛡️（2026-08）

#### 🛡️ 手势双重防御（真机双指缩放失效修复）

- Android `getMapAsync` 回调中显式调用 `setAllGesturesEnabled(true)` 兜底防御手势被意外禁用，并输出手势状态日志（`TDTMap: 手势状态 zoom=... scroll=...`）
- 组件搜索框 `.tdt-search-bar` 显式设置 `height: 70rpx` + `overflow: visible`，防止 absolute 容器被拉伸撑满后覆盖整幅地图、拦截地图手势

---

### v1.5.1 ⚙️（2026-08）

#### ⚙️ 新增纹理渲染模式

- 新增 `textureMode` prop（默认 false）：Android 端通过 `MapLibreMapOptions().textureMode(true)` 启用纹理渲染模式

---

### v1.5.0 🎉（2026-08）

#### 🆕 新增：H5 / Web 平台支持

- 新增 `utssdk/web/index.uts`（1087 行），基于 **MapLibre GL JS 4.7.1** 实现全部 58 个 API
- 组件 `tdt-map.uvue` 增加 `#ifdef WEB` 条件编译，App 端使用 `<native-view>`，Web 端使用 `<view>` 容器
- MapLibre GL JS 通过 unpkg CDN 在运行时动态加载（CSS + JS），无需额外构建配置
- `package.json` 开启 H5-mobile 和 H5-pc 所有主流浏览器支持
- `config.uts` 新增 `webApiKey` 字段，区分 App 端（服务端 Key）与 Web 端（浏览器端 Key）

**技术细节**：
| 模块 | Web 实现方式 |
|------|-------------|
| 底图 | `raster` source + XYZ `{x}{y}{z}` tile URL，8 子域名 |
| 覆盖物 | Marker（DOM）/ GeoJSON source + fill/line layer |
| 事件 | `click` / `mousedown`(长按) / `moveend`(相机) |
| 定位 | `navigator.geolocation`（需 HTTPS） |
| 截图 | `canvas.toDataURL('image/png')` → ArrayBuffer |
| 搜索 | `fetch()` → 天地图 REST API |
| 坐标转换 | 纯算法 GCJ02→WGS84（与 App 端完全一致） |

> ⚠️ **Web 端限制**：部分功能受浏览器环境制约——定位需 HTTPS、截图受跨域限制（Canvas taint）、TileJSON 等少数 API 不适用。

#### ⚡ 性能优化

- **Android 缩放/滑动卡顿修复**：style JSON 构建策略重写——从同时预建 6 个 RasterSource（vec/img/ter × 底图+注记）改为仅创建当前激活类型的 2 个 source，GPU 驻留瓦片数减少 67%，滑动显著流畅
- `switchMapType` 改为重建样式（`setStyle` + `fromJson`）替代旧的 visibility 切换，切换时间 <100ms

#### 🐛 Bug 修复

- 修复：Android `showUserLocation` 无定位蓝点图标——MapLibre `LocationComponent` 默认不渲染图标，现用 `GradientDrawable` 程序化生成蓝色定位圆点（填充 + 白色边框 + 半透明精度圈）
- 修复：Android 定位不生效——demo 页增加 `uni.authorize({ scope: 'scope.userLocation' })` 运行时权限请求，Android 6+ 必须先弹窗授权才能启用定位

---

### v1.4.0 🎉（2026-08）

#### 🆕 新增功能（6 项）

**地图交互增强**
- `setMapLongClickCallback(cb)` — 组件新增 `@mapLongClick` 事件，长按地图返回经纬度
- `setCameraChangeCallback(cb)` — 组件新增 `@cameraChange` 事件，视野移动完成时触发

**旋转与倾斜**
- `setBearing(bearing)` / `getBearing()` — 地图旋转（0°=正北，顺时针 0-360°）
- `setTilt(tilt)` / `getTilt()` — 地图倾斜（0°=垂直俯瞰，范围 0-60°）

**手势进阶**
- `enableRotate(enabled)` — 双指旋转手势开关
- `enableTilt(enabled)` — 双指倾斜手势开关

#### 技术实现
| 平台 | 旋转 (Bearing) | 倾斜 (Tilt) | 长按 | 相机事件 |
|------|---------------|-------------|------|---------|
| Android | `CameraPosition.Builder().bearing()` | `CameraPosition.Builder().tilt()` / `setMaxPitchPreference` | `addOnMapLongClickListener` | `addOnCameraIdleListener` |
| iOS | `MLNMapView.direction` | `MLNMapCamera.pitch` / `pitchEnabled` | `UILongPressGestureRecognizer` | `mapViewCameraDidChange` |
| 鸿蒙 | `CameraPosition.bearing` | `CameraPosition.tilt` / `setTiltGesturesEnabled` | `mapLongClick` 事件 | `cameraMove` + `cameraIdle` 事件 |

> ⚠️ **Android 倾斜手势限制**：MapLibre Native Android 的 `UiSettings` 不提供 `setTiltGesturesEnabled` 方法（与高德/Google Maps 不同），通过 `setMaxPitchPreference(0/60)` 限制倾斜范围实现等效效果。

---

### v1.2.0 ⚠️（2026-08）

#### 🆕 新增功能（15 项）

**覆盖物管理系统**
- `add*` 系列函数返回值改为 `number`（唯一 overlay ID）
- `removeOverlay(id)` — 按 ID 移除单个覆盖物
- `showOverlay(id)` — 按 ID 显示覆盖物
- `hideOverlay(id)` — 按 ID 隐藏覆盖物

**标记点击事件**
- `setMarkerClickCallback(cb)` — 组件新增 `@markertap` 事件

**地图操作增强**
- `checkResize()` — 触发布局重绘
- `planBy(x, y)` — 像素级平移
- `setViewport(points)` — 多点自适应视野

**信息计算**
- `getDistance(start, end)` — Haversine 距离计算（米）
- `getViewport(points)` — 计算最佳中心点与缩放级别
- `getSize()` — 获取容器像素尺寸
- `getCode()` — 返回坐标系 `EPSG:4326`

**天地图 Web API**
- `geocode_getLocation` — 逆地理编码（坐标 → 地址）
- `geocode_getPoint` — 地理编码（地址 → 坐标）
- `search / searchInBounds / searchNearby` — 关键字/范围/周边搜索

#### ⚠️ 破坏性变更
| 函数 | 旧返回值 | 新返回值 |
|------|---------|---------|
| `addMarker` | `void` | `number` |
| `addPolygon` | `void` | `number` |
| `addPolyline` | `void` | `number` |
| `addCircle` | `void` | `number` |
| `addRectangle` | `void` | `number` |
| `addLabel` | `void` | `number` |
| `setMarkers` | `void` | `Array<number>` |

---

### v1.1.0
- 新增折线/圆/矩形/文字标注/批量标记/清除全部覆盖物
- 手势控制（缩放/拖拽开关、最大/最小缩放限制）
- 视野边界限制
- GCJ02 → WGS84 坐标转换

### v1.0.0
- 初始版本
- 矢/影像/地形三图切换
- 标记点、多边形覆盖物
- 地图类型切换、截图、生命周期

---

## 接口调用说明

本插件涉及两类接口：

### 1. 地图瓦片（前端直接请求）
矢量/影像/地形瓦片由各平台原生引擎（MapLibre / MapKit）直接从天地图瓦片服务器拉取，**不经过任何后端**。Key 通过 `config.uts` 或组件 `apiKey` 属性传入，拼入瓦片 URL 中（App 端使用 `apiKey`，Web 端优先 `webApiKey`、未配置时 fallback `apiKey`）。

### 2. 天地图 Web API（UTS 原生层直接请求）
搜索、地理编码等 API 由 **UTS 原生层直接 HTTP 调用** `api.tianditu.gov.cn`，不走 uvue 前端运行时，也**不经过用户后端**。调用链路：

```
uvue 页面 → UTS 原生层 → HTTP → 天地图服务器
```

> ⚠️ **安全提示**：Key 会随 App 包体分发，存在泄露风险。生产环境建议通过**自有后端代理**天地图 API，App 调用自己的后端接口，由后端转发并注入 Key，避免 Key 暴露在客户端。

### 3. 后端代理示例（各语言）

**Key 选择说明（双 Key 场景）**：

- 代理模式下请求由**你的服务器**发给天地图，默认统一注入**服务端 Key**（`apiKey` 类型）即可，App 与 Web 客户端均可使用
- `webApiKey`（浏览器端 Key）仅用于 **Web 端直连天地图（不走代理）**；若 Web 端也走代理、且想按来源区分 Key（配额隔离），代理注入浏览器端 Key 时需把请求头 `Referer` 设置为你的 H5 站点域名（该域名需加入天地图控制台的浏览器端 Key 白名单），否则 Referer 校验失败
- 下方 Node.js 为**双 Key 版示例**（按客户端来源选择 Key），其余语言为单 Key 版（统一服务端 Key，如需区分来源参考 Node.js 示例）

#### Node.js (Express)（双 Key 版）

```js
const express = require('express')
const https = require('https')
const app = express()

const TDT_APP_KEY = '你的天地图服务端Key'      // App 端（原生）请求注入
const TDT_WEB_KEY = '你的天地图浏览器端Key'     // Web 端（H5）请求注入
const TDT_HOST = 'api.tianditu.gov.cn'

// 按客户端来源选择 Key：请求带 X-Client: web 时注入浏览器端 Key，否则注入服务端 Key
function pickKey(req) {
  return req.get('X-Client') === 'web' ? TDT_WEB_KEY : TDT_APP_KEY
}

app.get('/api/tdt/geocoder', (req, res) => {
  const params = new URLSearchParams(req.query)
  params.set('tk', pickKey(req))
  // 浏览器端 Key 需 Referer 白名单校验：注入 webApiKey 时把 Referer 设为你的 H5 站点域名
  const headers = { Referer: 'https://你的H5站点域名/' }
  https.get(`https://${TDT_HOST}/geocoder?${params}`, { headers }, (resp) => {
    let data = ''
    resp.on('data', chunk => data += chunk)
    resp.on('end', () => res.json(JSON.parse(data)))
  })
})

app.get('/api/tdt/search', (req, res) => {
  const params = new URLSearchParams(req.query)
  params.set('tk', pickKey(req))
  const headers = { Referer: 'https://你的H5站点域名/' }
  https.get(`https://${TDT_HOST}/search?${params}`, { headers }, (resp) => {
    let data = ''
    resp.on('data', chunk => data += chunk)
    resp.on('end', () => res.json(JSON.parse(data)))
  })
})
```

#### Python (Flask)

```python
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)
TDT_KEY = '你的天地图Key'
TDT_HOST = 'https://api.tianditu.gov.cn'

@app.route('/api/tdt/<path:endpoint>')
def proxy(endpoint):
    params = dict(request.args)
    params['tk'] = TDT_KEY
    r = requests.get(f'{TDT_HOST}/{endpoint}', params=params)
    return jsonify(r.json())
```

#### Java (Spring Boot)

```java
@RestController
@RequestMapping("/api/tdt")
public class TdtProxyController {

    private static final String TDT_KEY = "你的天地图Key";
    private static final String TDT_HOST = "https://api.tianditu.gov.cn";
    private final RestTemplate rest = new RestTemplate();

    @GetMapping("/{endpoint}")
    public Object proxy(@PathVariable String endpoint,
                        @RequestParam Map<String, String> params) {
        params.put("tk", TDT_KEY);
        String url = UriComponentsBuilder.fromHttpUrl(TDT_HOST + "/" + endpoint)
                .queryParams(params).toUriString();
        return rest.getForObject(url, Object.class);
    }
}
```

#### Go (net/http)

```go
func tdtProxy(w http.ResponseWriter, r *http.Request, endpoint string) {
    q := r.URL.Query()
    q.Set("tk", "你的天地图Key")
    url := fmt.Sprintf("https://api.tianditu.gov.cn/%s?%s", endpoint, q.Encode())
    resp, _ := http.Get(url)
    defer resp.Body.Close()
    body, _ := io.ReadAll(resp.Body)
    w.Header().Set("Content-Type", "application/json")
    w.Write(body)
}
```

#### PHP

```php
<?php
define('TDT_KEY', '你的天地图Key');
define('TDT_HOST', 'https://api.tianditu.gov.cn');

$endpoint = $_GET['_endpoint'] ?? 'geocoder';
unset($_GET['_endpoint']);
$_GET['tk'] = TDT_KEY;

$url = TDT_HOST . '/' . $endpoint . '?' . http_build_query($_GET);
echo file_get_contents($url);
```

> App 端调用改为你自己的后端地址，后续如果 Key 泄露只需在后端轮换，无需重新发版。

---

## 更新日志

### v1.9.0 🖼️（最新）

- ✨ 新增：图片覆盖物 `addImageOverlay` / `updateImageOverlay` / `@imageOverlayClick`——图片按地理范围拉伸贴合，支持 transparency/zIndex 三档/bearing 旋转/visible/clickable，复用 overlay ID 统一管理体系
- 🌐 四端统一：Android（ImageSource URL 直载 / 本机路径 io 线程两段解码防 OOM）、iOS（MLNImageSource 引擎直载 / 本机 UIImage）、鸿蒙（华为原生 addImageOverlay 全参数 + http 下载转 PixelMap + >4096 下采样）、Web（image source + raster layer）
- 🖱️ 点击检测：MapLibre 三端统一几何 hit-test（raster 图层无要素），命中互斥消费 mapClick；鸿蒙原生 imageOverlayClick
- 🔁 setStyle 后图片覆盖物随用户图层自动重建（MapLibre 三端，同 ID 保序）

### v1.6.0 🎛️

- ✨ 新增：缩放按钮 / 比例尺 / 版权条基础控件组（Android/iOS 为 uvue 层，鸿蒙/Web 为原生控件）
- ✨ 新增 props：`showZoomControl` / `showScaleControl` / `showCopyrightControl` / `copyrightText`
- 🐛 修复：Android 缩放按钮 `animateCamera` 动画不推进——改用 `moveCamera`
- 🐛 修复：隐藏 MapLibre 原生 logo/attribution（Android/iOS/Web），鸿蒙华为 logo 移位右上避让

### v1.5.8 🔬

- 🔬 扩展：相机移动统计覆盖全部来源（reason 1=手势 / 2=开发者动画 / 3=API 动画含 fling 惯性）

### v1.5.7 🔬

- 🔬 新增：Android 性能诊断日志 `[PERF]`——手势逻辑帧 + 渲染帧 + GPU 耗时统计，cameraIdle 汇总输出

### v1.5.6 🎉

- ✨ 新增：marker 气泡 `showBubble`/`hideBubble`——uvue 自定义气泡 + Web Mercator 换算，随地图移动/缩放自动跟随
- 🐛 修复：气泡坐标四端单位不一致——改用 `getBoundingClientRect` 容器自测（四端统一 css px）
- 🐛 修复：跨运行时 `UTSJSONObject` 下标访问失败——`JSON.parse` 本地重建
- 🐛 修复：天地图 v2 搜索未传 `count` 参数

### v1.5.5 ⚡

- ⚡ 优化：Android 双指缩放掉帧——`pixelRatio` 限制 2.0 + `setPrefetchZoomDelta(2)`
- 🐛 修复：`animateCamera` 动画不推进——改用 `moveCamera` 瞬时跳转
- 🐛 修复：搜索结果列表点击穿透到地图——列表改为独立容器

### v1.5.4 🐛

- 🐛 修复：搜索/地点搜索解析失败——天地图老搜索接口已下线，四端迁移「地名搜索2.0」`/v2/search`，JSON.parse 判空防御
- 🐛 修复：Android 定位按钮无反应——运行时权限申请、定位服务检测、GPS/网络双通道并发定位
- ⚡ 优化：Android 滑动卡顿——`raster-fade-duration: 0` 消除瓦片淡入模糊

### v1.5.3 🐛

- 🐛 修复：定位按钮 `uni.authorize is not a function`——uni-app x 无此 API，移除后直接调定位（首次自动弹授权框，推翻 v1.5.0 的 uni.authorize 方案）
- 🐛 修复：`回调函数已释放，不能再次执行`——四端 10 个带回调导出函数全部添加 `@UTSJS.keepAlive` 装饰器
- 🐛 修复：`HTTP 请求失败: null`——Android 主线程同步 HttpURLConnection 改 `uni.request` 异步，并修复 `res.data` 类型处理
- 🐛 修复：Web 端 3 处内联类型字面量 → 命名类型（MarkerInfo/TileLayerInfo/TdtLayerConfig）

### v1.5.2 🛡️

- 🛡️ 手势兜底：Android 显式 `setAllGesturesEnabled(true)` + 手势状态日志
- 🐛 修复：搜索框 absolute 容器撑满拦截地图手势（`height:70rpx` + `overflow:visible`）

### v1.5.1 ⚙️

- ✨ 新增：`textureMode` prop（Android 纹理渲染模式）

### v1.5.0 🎉

- ✨ 新增：**H5 / Web 平台完整支持**——新增 `utssdk/web/index.uts`（1087 行），基于 MapLibre GL JS 4.7.1 实现全部 58 个 API
- ✨ 新增：**Web Key 配置**——`config.uts` 新增 `webApiKey` 字段，区分 App 端（服务端/Android 平台类型）与 Web 端（浏览器端类型）
- ✨ 新增：组件 `tdt-map.uvue` 增加 `#ifdef WEB` 条件编译，App/Web 双路径无冲突
- ⚡ 优化：**Android 缩放/滑动卡顿修复**——style JSON 从 6 RasterSource 减至 2 个，GPU 瓦片驻留减少 67%
- ⚡ 优化：`switchMapType` 改为重建样式替代 visibility 切换，避免不可见 source 持续占用资源
- 🐛 修复：Android `showUserLocation` 无定位蓝点图标——`GradientDrawable` 程序化生成蓝色定位圆点
- 🐛 修复：Android 定位不生效——demo 页增加 `uni.authorize` 运行时权限请求

### v1.4.0 🎉

- ✨ 新增：地图交互增强——`@mapLongClick` 长按事件、`@cameraChange` 相机视野变化完成事件
- ✨ 新增：旋转与倾斜——`setBearing` / `getBearing` / `setTilt` / `getTilt`，bearing 旋转（0-360°）、tilt 倾斜（0-60°）
- ✨ 新增：手势进阶——`enableRotate` 双指旋转手势开关、`enableTilt` 双指倾斜手势开关
- 🐛 修复：Android `UiSettings.setTiltGesturesEnabled` 不存在导致运行时崩溃（改用 `setMaxPitchPreference`）
- 🐛 修复：鸿蒙 `cameraChange` 事件不存在导致回调永不触发（改用 `cameraMove` + `cameraIdle`）
- 🐛 修复：iOS `UILongPressGestureRecognizer` 无 state 过滤导致单次长按多次触发
- 🐛 修复：iOS `mapView.camera` nil 防御加固

### v1.3.0 🎉

- ✨ 新增：定位能力——`showUserLocation` 显示/隐藏用户位置蓝点，`getUserLocation` 获取当前 GPS 坐标（Android `LocationManager` / iOS `CLLocationManager` / 鸿蒙 `geoLocationManager`）
- ✨ 新增：信息窗口——`showMarkerInfo` / `hideMarkerInfo` 控制标记气泡弹窗，`addMarker` 增加 `snippet` 参数支持副标题
- ✨ 新增：自定义瓦片图层——`addCustomTileLayer` / `removeCustomTileLayer` 叠加外部 XYZ 瓦片（如 OSM、天气、交通专题图），Android `RasterSource+RasterLayer` / iOS `MLNRasterTileSource` / 鸿蒙 `addTileOverlay`
- 🐛 修复：Android `searchNearby` 函数缺失导致调用方报 undefined 错误
- 🐛 修复：三端 `destroyMap` 内存泄漏（Android 4 项 / iOS 10 项 / 鸿蒙 6 项状态变量未清理）
- 🐛 修复：iOS `getUserLocation` 未请求定位授权，iOS 14+ 定位永远静默失败
- 🐛 修复：Android `getUserLocation` 无超时机制，GPS 关闭时回调永不触发
- 🐛 修复：Android `activateLocationComponentIfNeeded` / `showMarkerInfo` NPE 风险（style 未加载或 mapLibreMap 为空时崩溃）
- 🐛 修复：Android 标记点击回调永不触发——`OnMapClickListener` 返回 `true` 消费所有事件
- 🐛 修复：iOS 标记点击崩溃——`@objc` 方法参数名 `annotation` 应为 `didSelectAnnotation`
- 🐛 修复：iOS `destroyMap` 未 invalidate 定位轮询 NSTimer，销毁后仍持续运行
- 🐛 修复：Android `removeOverlay` 未清理 `markerList`/`polygonList` 等类型列表，后续 `clearMarkers` 重复操作已移除对象
- 🐛 修复：鸿蒙 `clearOverlays` 未同步清理 `customTileOverlays` Map
- 🐛 修复：鸿蒙 `addRectangle`/`setMarkers`/`clearOverlays`/`switchMapType` 声明为 `async` 返回 `Promise`，导致 overlay ID 追踪断裂
- 🐛 修复：鸿蒙自定义 `urlEncode` 用 UCS-2 编码中文（非 UTF-8），搜索含中文时 API 返回参数错误
- 🐛 修复：三端天地图 Web API 统一改用 HTTPS（Android 9+ 默认拦截 HTTP 明文流量）

### v1.2.1

- 🐛 修复：HarmonyOS `setMarkerClickCallback` 重复注册 `markerClick` 监听器导致 `@markertap` 事件双重触发

### v1.2.0

- ✨ 新增：15 项功能——折线/圆/矩形覆盖物、文字标注、批量标记、覆盖物 ID 追踪系统（`removeOverlay`/`showOverlay`/`hideOverlay`）、标记点击事件 `@markertap`、`zoomIn`/`zoomOut`/`setMinZoom`/`setMaxZoom`、`panTo`/`centerAndZoom`、`enableZoom`/`enableDrag`、`setMaxBounds`、`getBounds`、GCJ02→WGS84 坐标转换、`checkResize`/`planBy`/`setViewport`/`getSize`、Haversine 测距/最佳视野计算/坐标系查询、天地图地理编码/搜索 API

### v1.1.0

- ✨ 新增：地图类型切换、标记/多边形基础覆盖物、`fitBounds`/`getCenter`/`getZoom`、`snapshot`

### v1.0.0

- 🎉 初始版本：天地图矢量/影像/地形底图（MapLibre Android/iOS + 华为 MapKit 鸿蒙），三端统一 API

---

## 常见问题

**Q: 瓦片不显示（灰白背景）？**  
A: 99% 是 Key 类型错误。请到 [天地图控制台](https://console.tianditu.gov.cn) 创建「服务端」或「Android 平台」类型 Key，不要使用浏览器端 Key。

**Q: HarmonyOS 截图报错？**  
A: 华为 MapKit 不支持 API 截图，`snapshot(callback)` 回调空 `ArrayBuffer` 并上报错误。

**Q: iOS 编译报 MLNMapView 找不到？**  
A: 确保 `config.json` 或 `package.json` 中已声明 MapLibre iOS SDK 依赖。
