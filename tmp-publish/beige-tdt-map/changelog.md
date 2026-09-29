# beige-tdt-map 更新日志

## 1.9.0（发布时填写日期）

- ✨ 新增：图片覆盖物 `addImageOverlay` / `updateImageOverlay` / `setImageOverlayClickCallback`（组件层 `@imageOverlayClick`）——图片按地理范围拉伸贴合，支持 transparency/zIndex 三档/bearing 旋转/visible/clickable，复用 overlay ID 统一管理体系（removeOverlay/showOverlay/hideOverlay/clearOverlays）
- 🌐 四端统一：Android（ImageSource URL 直载 / 本机路径 io 线程两段解码防 OOM）、iOS（MLNImageSource 引擎直载 / 本机 UIImage 同步读）、鸿蒙（华为原生 addImageOverlay 全参数直通 + http 下载转 PixelMap + >4096 下采样）、Web（image source + raster layer）
- 🖱️ 点击检测：MapLibre 三端统一几何 hit-test（raster 图层无要素，不用 queryRenderedFeatures），命中互斥消费 mapClick；鸿蒙原生 imageOverlayClick（getId 反查插件 ID）
- 🖼️ 图片来源：http(s) URL 与本机缓存绝对路径（缓存由调用方维护）；Web 端仅 http(s)/data: URL，本机路径上报 code=4
- 🔁 setStyle 后图片覆盖物随用户图层自动重建（MapLibre 三端，同 ID 保序）；异步加载中 update 合并应用、失败上报 code=2/4 且 ID 失效

## 1.8.0（发布时填写日期）

- ✨ 新增：`setStyle` / `removeStyle`——运行时替换/还原 MapLibre 样式（style spec JSON，须含 version 字段），切换后自动重建用户图层（自定义瓦片/WMS/聚合/热力），天底图由自定义样式自带
- ✨ 新增：`addWMSTileLayer`——WMS 服务叠加，自动拼装 GetMap 请求（Android/iOS/Web；鸿蒙上报 code=5）
- ✨ 新增：`addCustomTileLayer`/`addTileLayer` 支持 options（opacity/minZoom/maxZoom/zIndex，zIndex 仅支持 0 与缺省）
- ✨ 新增：`setMaxBounds`/`setMapMaxBounds` 参数可空——传 null 清除范围限制
- ✨ 新增：`isSupportCanvas`——平台 canvas 标注能力探测（Web=true；其余 false）
- 🐛 修复：iOS setStyle 后天地图瓦片叠影——delegate 增加三态拦截（styleRebuildMode），setStyle 后不再无条件重建天底图
- ⚠️ 鸿蒙：`setStyle`/`addWMSTileLayer` 显式上报 mapError code=5（平台不支持，不静默失败）；options 仅 transparency（opacity）生效

## 1.7.0（发布时填写日期）

- ✨ 新增：标记聚合 `addMarkerCluster` / `removeMarkerCluster`——海量 POI 自动聚合显示，聚合簇带数量徽标，点击自动放大展开（Android/iOS/Web 走 MapLibre GeoJSON cluster，鸿蒙走华为原生 addClusterOverlay）
- ✨ 新增：热力图 `addHeatmap` / `removeHeatmap`——密度数据可视化，支持权重、半径、透明度配置（Android/iOS/Web 走 MapLibre HeatmapLayer，鸿蒙走华为原生 addHeatmap，需 HarmonyOS 6.0+ 低版本显式报错）
- 🐛 修复：聚合数字标签不渲染——空 style 无 glyphs 时 text-field 图层被 style spec 强制校验静默拒绝，三端 style 补 demotiles 字体端点 + 显式 text-font（浏览器实测发现）
- 📝 文档：README 新增 v1.7.0 API 说明与示例

## 1.6.0（2026-08-16）

- ✨ 新增：缩放按钮控件——Android/iOS 为 uvue 层样式（调 zoomIn/zoomOut），鸿蒙为原生 zoomControlsEnabled，Web 为原生 NavigationControl
- ✨ 新增：比例尺控件——Android/iOS 为 uvue 层公式计算（OSM Wiki ground resolution），鸿蒙为原生 scaleControlsEnabled，Web 为原生 ScaleControl
- ✨ 新增：版权条控件——四端统一 uvue 层（天地图服务条款合规要求），默认"© 天地图"，可通过 copyrightText 填入审图号
- ✨ 新增 props：showZoomControl / showScaleControl / showCopyrightControl / copyrightText
- 📌 注意：showZoomControl/showScaleControl 在鸿蒙/Web 原生端为初始化一次性配置（同 textureMode 限制）；Android/iOS 的 uvue 控件与版权条 props 天然响应式
- 🐛 修复：Android/iOS/Web 隐藏 MapLibre 原生 logo 与 attribution（避免与版权条重叠及错误版权信息）；鸿蒙华为 logo 移位右下角避让
- 🐛 修复：鸿蒙 logo 位置 TOP_END→BOTTOM_END——右上角与华为默认指南针（旋转/倾斜时出现）重叠；BOTTOM_END 为华为官方示例位置，与缩放控件共存
- 🐛 修复：Android 缩放按钮无响应——`zoomIn`/`zoomOut` 沿用 v1.5.5 实测结论改用 `moveCamera` 瞬时缩放（`animateCamera` 动画在 UTS 集成下不推进）

## 1.5.8（2026-08）

- 🔬 扩展：相机移动性能统计覆盖全部来源——reason 1=手势 / 2=开发者动画 / 3=API 动画（含 fling 惯性），用于排查松手后惯性滑动是否触发与推进

## 1.5.7（2026-08）

- 🔬 新增：Android 性能诊断日志（`TDTMap: [PERF]`）——手势逻辑帧统计（cameraMoveStarted/cameraMove 计数 + cameraIdle 汇总输出，避免逐帧日志加剧卡顿）+ 渲染帧统计（onDidFinishRenderingFrame 真实画面帧 + GPU 编码/渲染耗时，对比逻辑帧定位「逻辑快画面卡」）

## 1.5.6（2026-08）

- ✨ 新增：marker 气泡（`showBubble`/`hideBubble`）——MapLibre 11.x 原生 InfoWindow 在 native-view 集成下不显示，改用 uvue 自定义气泡 + Web Mercator 数学换算，随地图移动/缩放自动跟随，点击气泡关闭
- 🐛 修复：气泡坐标四端单位不一致——各端 UTS `getSize` 单位各异（Android 物理像素 / iOS point / 鸿蒙恒 `{0,0}` / H5 css px），改用 `uni.getElementById().getBoundingClientRect()` 容器自测（四端统一 css px）
- 🐛 修复：跨运行时 `UTSJSONObject` 下标访问失败——`JSON.parse` 本地重建后使用 `[]` 下标访问
- 🐛 修复：天地图 v2 搜索未传 `count` 参数——`search`/`searchInBounds`/`searchNearby` 统一透传

## 1.5.5（2026-08）

- ⚡ 优化：Android 双指缩放掉帧——`pixelRatio` 限制 2.0 + `setPrefetchZoomDelta(2)` 预取相邻层级瓦片
- 🐛 修复：`animateCamera` 动画不推进（日志实测 500ms 零进展）——`panTo`/`centerAndZoom`/选中搜索结果定位改用 `moveCamera` 瞬时跳转
- 🐛 修复：搜索结果列表点击穿透到地图——溢出子元素 hit-test 失效，列表改为独立容器绝对定位

## 1.5.4（2026-08）

- 🐛 修复：搜索/地点搜索解析失败——天地图老搜索接口已下线，四端迁移「地名搜索2.0」`/v2/search`，JSON.parse 判空防御
- 🐛 修复：Android 定位按钮无反应——运行时权限申请、定位服务开关检测、GPS/网络双通道并发定位（先回先得）
- ⚡ 优化：Android 滑动卡顿——`raster-fade-duration: 0` 消除瓦片淡入模糊

## 1.5.3（2026-08-13）

- 🐛 修复：定位按钮 `uni.authorize is not a function`——uni-app x 无此 API，移除后直接调定位（首次自动弹授权框，推翻 v1.5.0 的 uni.authorize 方案）
- 🐛 修复：`回调函数已释放，不能再次执行`——四端 10 个带回调导出函数全部添加 `@UTSJS.keepAlive` 装饰器
- 🐛 修复：`HTTP 请求失败: null`——Android 主线程同步 HttpURLConnection 改 `uni.request` 异步，并修复 `res.data` 类型处理
- 🐛 修复：Web 端 3 处内联类型字面量 → 命名类型（MarkerInfo/TileLayerInfo/TdtLayerConfig）

## 1.5.2（2026-08）

- 🛡️ 手势兜底：Android 显式 `setAllGesturesEnabled(true)` + 手势状态日志
- 🐛 修复：搜索框 absolute 容器撑满拦截地图手势（`height:70rpx` + `overflow:visible`）

## 1.5.1（2026-08）

- ✨ 新增：`textureMode` prop——Android 纹理渲染模式（模拟器/特殊层叠场景启用，MapLibre 官方 issue #3070 workaround）

## 1.5.0（2026-08）

- ✨ 新增：H5 / Web 平台完整支持——新增 `utssdk/web/index.uts`，基于 MapLibre GL JS 4.7.1 实现全部 API
- ✨ 新增：Web Key 配置——`config.uts` 新增 `webApiKey` 字段（浏览器端 Key 与 App 端服务端 Key 分离）
- ✨ 新增：组件内置搜索框、内置定位按钮（`showSearchBar`/`showLocateButton` 开关）
- ⚡ 优化：Android 缩放/滑动卡顿——style JSON 从 6 RasterSource 减至 2 个，GPU 瓦片驻留减少 67%
- ⚡ 优化：`switchMapType` 改为重建样式替代 visibility 切换
- 🐛 修复：Android 定位蓝点不显示——`GradientDrawable` 程序化生成蓝点（填充+白边+精度圈）
- 🐛 修复：Android 定位不生效——demo 页增加运行时权限请求

## 1.4.0（2026-08）

- ✨ 新增：地图交互增强——`@mapLongClick` 长按事件、`@cameraChange` 相机视野变化完成事件
- ✨ 新增：旋转与倾斜——`setBearing`/`getBearing`/`setTilt`/`getTilt`（bearing 0-360°、tilt 0-60°）
- ✨ 新增：手势进阶——`enableRotate` 双指旋转开关、`enableTilt` 双指倾斜开关
- 🐛 修复：Android `UiSettings.setTiltGesturesEnabled` 不存在导致运行时崩溃（改用 `setMaxPitchPreference`）
- 🐛 修复：鸿蒙 `cameraChange` 事件不存在导致回调永不触发（改用 `cameraMove` + `cameraIdle`）
- 🐛 修复：iOS `UILongPressGestureRecognizer` 无 state 过滤导致单次长按多次触发
- 🐛 修复：iOS `mapView.camera` nil 防御加固

## 1.3.0（2026-08）

- ✨ 新增：定位能力——`showUserLocation`/`getUserLocation`（Android LocationManager / iOS CLLocationManager / 鸿蒙 geoLocationManager）
- ✨ 新增：信息窗口——`showMarkerInfo`/`hideMarkerInfo`，`addMarker` 增加 `snippet` 参数
- ✨ 新增：自定义瓦片图层——`addCustomTileLayer`/`removeCustomTileLayer`
- 🐛 修复：Android `searchNearby` 函数缺失导致调用方报 undefined
- 🐛 修复：三端 `destroyMap` 内存泄漏（Android 4 项 / iOS 10 项 / 鸿蒙 6 项状态变量未清理）
- 🐛 修复：iOS `getUserLocation` 未请求定位授权，iOS 14+ 定位永远静默失败
- 🐛 修复：Android `getUserLocation` 无超时机制，GPS 关闭时回调永不触发
- 🐛 修复：Android `activateLocationComponentIfNeeded`/`showMarkerInfo` NPE 风险
- 🐛 修复：Android 标记点击回调永不触发——`OnMapClickListener` 返回 true 消费所有事件
- 🐛 修复：iOS 标记点击崩溃——`@objc` 参数名应为 `didSelectAnnotation`
- 🐛 修复：iOS `destroyMap` 未 invalidate 定位轮询 NSTimer
- 🐛 修复：Android `removeOverlay` 未清理类型列表导致重复操作已移除对象
- 🐛 修复：鸿蒙 `clearOverlays` 未同步清理 `customTileOverlays` Map
- 🐛 修复：鸿蒙 async 函数返回 Promise 导致 overlay ID 追踪断裂
- 🐛 修复：鸿蒙自定义 `urlEncode` 用 UCS-2 编码中文（非 UTF-8），搜索含中文参数错误
- 🐛 修复：三端天地图 Web API 统一改用 HTTPS（Android 9+ 默认拦截 HTTP 明文流量）

## 1.2.1

- 🐛 修复：HarmonyOS `setMarkerClickCallback` 重复注册 `markerClick` 监听器导致 `@markertap` 事件双重触发

## 1.2.0（2026-08）

- ✨ 新增：15 项功能——折线/圆/矩形覆盖物、文字标注、批量标记、覆盖物 ID 追踪系统（removeOverlay/showOverlay/hideOverlay）、标记点击事件 `@markertap`、zoomIn/zoomOut/setMinZoom/setMaxZoom、panTo/centerAndZoom、enableZoom/enableDrag、setMaxBounds/getBounds、GCJ02→WGS84 坐标转换、checkResize/planBy/setViewport/getSize、Haversine 测距/最佳视野计算/坐标系查询、天地图地理编码/搜索 API
- ⚠️ 破坏性变更：`addMarker`/`addPolygon`/`addPolyline`/`addCircle`/`addRectangle`/`addLabel` 返回值 `void` → `number`（overlay ID），`setMarkers` 返回 `Array<number>`

## 1.1.0

- ✨ 新增：地图类型切换、标记/多边形基础覆盖物、`fitBounds`/`getCenter`/`getZoom`、`snapshot`

## 1.0.0

- 🎉 初始版本：天地图矢量/影像/地形底图（MapLibre Android/iOS + 华为 MapKit 鸿蒙），三端统一 API
