# Android 端 annotations API → GeoJsonSource + Layer 体系迁移设计

> 版本基线：v1.9.3（maplibre-android-sdk 11.8.1），目标版本 v1.10.0
> 范围：仅 `utssdk/app-android/index.uts`（2954 行）；interface.uts 与 tdt-map.uvue 零改动
> 已确认决策：SymbolLayer+addImage 渲染 marker；showMarkerInfo/hideMarkerInfo no-op；label 走文字位图；子模块 feature 分支提交后主仓 `make fe-sync-uts-map` 同步

---

## 0. 迁移总览

弃用 API 全部清除（`MarkerOptions/Marker/PolygonOptions/Polygon/PolylineOptions/Polyline` 及 `MapLibreMap.addMarker/removeMarker/addPolygon/removePolygon/addPolyline/removePolyline`），六类覆盖物统一落 style 层系（source+layer），复用项目内已验证的四条先例通路：

| 通路 | 先例 | 复用点 |
|------|------|--------|
| GeoJSON 字符串直构 source | `addStyledPolylineBatch` L719-741 | `new GeoJsonSource(id, json)` 绕开嵌套泛型 |
| LineLayer 样式 | `addStyledPolyline` L677-689 | lineColor/lineWidth/lineOpacity/lineDasharray |
| visibility 显隐 | showOverlay/hideOverlay L1432/1468 | `PropertyFactory.visibility(VISIBLE/NONE)` |
| 重放快照→重建→恢复可见性 | `rebuildUserLayersAfterStyle` L2602-2634 | 图片覆盖物三段式 |

## 1. 数据结构设计

**删除**五个 legacy List（L65-69：markerList/polygonList/polylineList/circleList/rectangleList）。

**新增三组快照数组 + 两个共享层引用**（marker/label 因 `setMarkers` 批量与视觉同质走共享层；shape 四类异构走独立层）：

```uts
// ── marker/label：共享 source+layer，快照数组为唯一事实源 ──
type MarkerSnapshot = { oid: number, lat: number, lng: number, title: string, snippet: string, hidden: boolean }
type LabelSnapshot = { oid: number, lat: number, lng: number, text: string, hidden: boolean }
let markerSnapshots: Array<MarkerSnapshot> = []
let labelSnapshots: Array<LabelSnapshot> = []
let markerSourceRef: GeoJsonSource | null = null
let markerLayerRef: SymbolLayer | null = null
let markerBitmapRegistered: boolean = false   // 当前 style 生命周期内图钉位图是否已注册

// ── shape 四类：独立 source+layer，快照供重放 ──
type ShapeOverlayRef = { source: any, layers: Array<any> }   // polygon/circle/rectangle: [fill, line]; polyline: [line]
type VectorSnapshot = {
    oid: number, kind: string,          // 'polygon'|'polyline'|'circle'|'rectangle'
    points: Array<MapClickDetail>,      // circle 已展开为 36 点（circleToPolygonPoints 不变，L890）
    color: string, fillColor: string, width: number, visible: boolean
}
let vectorSnapshots: Array<VectorSnapshot> = []
```

**overlayMap 合并方式**：`OverlayEntry`（L76-81）结构不动。marker/label 的 `nativeRef` 存 null（共享层由模块级 ref 持有，snapshot 数组即状态源，`allocateOverlayId('marker', null)` 合法——L861 先例）；shape 的 `nativeRef` 存 `ShapeOverlayRef`（对齐 image 通路 `{source,layer}` 包装注释 L79）。`entry.type` 判等字符串不变（marker/label/polygon/polyline/circle/rectangle），show/hide/remove 按 type 分派。

跨类 z-order 由固定层序规则保证（§9），无需恢复全局添加序，数组内序即可。

## 2. marker 通路

**图钉位图**（Canvas 自绘，约 30 行，新增 `import { Path } from "android.graphics"`）：

```uts
const MARKER_ICON_NAME = "tdt-marker-icon"
function buildDefaultPinBitmap(density: number): Bitmap {
    // 36dp 盒子；红色水滴 = drawCircle 圆头(18,13,r=10dp) + Path 三角尾(8,13)→(18,34)→(28,13)，同色拼合
    // 内加 5dp 白色内圆增强视觉；物理像素 = dp * density（对齐 addStyledText L817 先例）
}
```

**注册时机**：`ensureMarkerStyle(style)` 内聚三件事（幂等，addMarker/setMarkers/重放统一入口）：
1. `markerBitmapRegistered == false` → `style.addImage(MARKER_ICON_NAME, bmp)`，置 true；
2. `markerLayerRef == null`（当前 style 未建层）→ 建 source（空 FeatureCollection）+ SymbolLayer，锚定层序（§9）；
3. 位图引用由 style 持有，模块级只存 `markerBitmapRegistered` 标志——**setStyle 后 style 的 image 表清空，重放路径置 false 重注册**。

**SymbolLayer 属性**：

```uts
layer.setProperties(
    PropertyFactory.iconImage(MARKER_ICON_NAME),
    PropertyFactory.iconSize(1.0.toFloat()),          // 位图已按 density 物理像素绘制，size=1；编译验证项
    PropertyFactory.iconAllowOverlap(true),           // legacy annotation 无碰撞隐藏，必须 true 保持行为
    PropertyFactory.iconAnchor("bottom")              // 针尖对准 lat/lng = legacy Marker 默认锚点
)
```

**数据写入**：`rebuildMarkerSource()` 把 markerSnapshots（hidden=false 的）全量构 FeatureCollection（每个 Feature 的 properties 存 `oid/lat/lng/title/snippet`，JsonObject addProperty，L2913 先例）→ `markerSourceRef.setGeoJson(...)`。增删改单点 = 改数组 + 一次全量重放（百级无压力，聚合同款数据量级先例）。

**addMarker 流程**：`ensureMarkerStyle` → push snapshot → `rebuildMarkerSource()` → `allocateOverlayId('marker', null)`。setMarkers：clearMarkers + 循环 push + 一次 rebuild（共享层只建一次，批量性能天然优于 legacy 逐个 addMarker）。

## 3. label 通路（推荐方案：b — 文字转位图 + SymbolLayer icon-image）

**理由**：
- glyphs 约束下 SymbolLayer `text-field` 大概率静默失败（GLYPHS_ENDPOINT 指 demotiles，L150/L787 已确证）；
- 候选 a（ImageSource+RasterLayer，addStyledText 同款）是**地理 bounds** 语义：文字随缩放变大小、需按当前 zoom 换算 halfLng/halfLat（L844-852），作"标注"体验差（缩放后文字模糊/变形）；
- 候选 b 是**屏幕像素**语义：icon 恒定像素尺寸、渲染管线与 marker 共用（同一套 addImage+SymbolLayer），增量代码仅一个 `buildLabelBitmap`；
- **行为修正说明**：legacy Android 11.x 下 label 的 title 仅存在于 InfoWindow，而 InfoWindow native-view 缺陷实测不显示（L15-17 组件层注释）→ legacy 文字实际不可见。接口零改动约束下（markertap 只回 {lat,lng}，无 title 通路），常显文字位图是唯一让"文字标注"语义成立的路径。此为有意的行为修正，非等价迁移。

**实现**：`buildLabelBitmap(text, oid, density)` —— 复用 addStyledText 的 Canvas 绘制内核（Paint measureText/FontMetrics/drawText，L821-842），固定样式：14sp 黑字 + 2dp 白 halo（提高底图对比度）；注册名 `"tdt-label-icon-" + oid`。共享层 `tdt-label-layer` 的 `iconImage(Expression.get("img"))`（feature properties 存位图名，逐 feature 换图）；`iconAnchor("center")`、`iconAllowOverlap(true)`。每个 label 一张位图，重放时按 snapshot 重画重注册（位图不缓存持有，内存友好；label 数量为标注级非帧图级）。

label 同样进 P1 点击命中（type='label'，回 {lat,lng}=label 坐标，与 legacy markerList 遍历含 label 一致）。

## 4. polygon / circle / rectangle 通路

**双层结构**（每 overlay 一个 source + fill/line 两层）：

```uts
const sourceId = "tdt-shape-src-" + id      // GeoJsonSource(id, geojsonString) 直构
const fillId   = "tdt-shape-fill-" + id     // FillLayer: fill-color(纯RGB) + fill-opacity(原色alpha)
const lineId   = "tdt-shape-line-" + id     // LineLayer: line-color/line-opacity 同拆 + line-width
```

**颜色语义对齐 legacy**：`Color.parseColor` 是 `#aarrggbb`（alpha 在前，demo `'#FF660044'` L299 实证）。新增拆分函数：

```uts
function splitColorAlpha(argb: string): { colorInt: number, alpha: number } {
    const c = Color.parseColor(argb)            // 返回值可为负（L793 注释），掩码运算正确
    return { colorInt: (c & 0xFFFFFF) | 0xFF000000, alpha: ((c >> 24) & 0xFF) / 255.0 }
}
```

- `fill-color` = fillColor 的 RGB、`fill-opacity` = fillColor 的 alpha（默认不透明时 1.0）；
- `line-color`/`line-opacity` 同拆 color（stroke 参数）；`line-width` = **5f 对齐 PolygonOptions 默认 stroke width**（编译后真机目测微调，验证点）；
- circle 保持 36 边近似（`circleToPolygonPoints` 原样复用后存入 snapshot.points）；rectangle 四点入 snapshot.points；
- geometry JSON 字符串直构（fill 渲染自动闭环，无需首尾重复点）：`'{"type":"Polygon","coordinates":[' + buildRingJson(points) + ']}'`；`buildRingJson` 输出 `[[lng,lat],[lng,lat],...]`（batch 先例 L730-734 同款 `toDouble().toString()` 拼接）。

**统一入口** `addShapeOverlay(kind, oid, points, color, fillColor, width)`：建 source → fill 层 → line 层（fill 先 add 保证 line 盖 fill，锚定 §9）→ `overlayMap.set(id, {type: kind, nativeRef: {source, layers:[fill,line]}, isVisible: true})` → push vectorSnapshot。addPolygon/addCircle/addRectangle 各自算好 points 后调它（对齐 legacy 错误上报 code=4 行为）。

## 5. polyline 通路（与 addStyledPolyline 的关系）

**参数对齐、内核复用、管理分离**：legacy addPolyline = `addStyledPolyline(points, color, width, dash=null, opacity=1.0)` 的样式语义。提取私有 helper：

```uts
function buildLineSourceLayer(sourceId: string, layerId: string, json: string,
                              color: string, width: number, dash: Array<number> | null, opacity: number): ShapeOverlayRef
```

`addStyledPolyline/addStyledPolylineBatch` 改调 helper（行为零变化，仍走 styledPolylines 数组 + type='styledPolyline'）；`addPolyline` 调 helper（dash=null, opacity=1.0，LineString 单环直构）+ 走 vectorSnapshots + type='polyline'。两套管理面并存：styled 保持既有不重放行为（历史现状，不扩范围）；polyline 纳入六类重放（§8，行为增强，annotation 时代 setStyle 后同样丢失，无回归风险）。

## 6. 点击命中改造（initMap P1 块，替换 L316-334）

```uts
// Priority 1: marker/label 点击（queryRenderedFeatures 矩形容差命中，替代 markerList 欧氏遍历）
const markerCb = markerClickCallback
if (markerCb != null && (markerLayerRef != null || labelLayerRef != null)) {
    const screenPt = map.getProjection().toScreenLocation(point)   // L297 先例
    const tol = 20.0                                              // px 容差，常量 TAP_TOLERANCE_PX
    const rect = new RectF(screenPt.x - tol, screenPt.y - tol, screenPt.x + tol, screenPt.y + tol)
    // vararg 多层 id：label 层在前（视觉在上者优先命中）；单参先例 L301，多参为编译期验证项
    const hitRaw = map.queryRenderedFeatures(rect, "tdt-label-layer", "tdt-marker-layer")
    if (hitRaw != null) {
        const hitFeatures = Array.fromNative(hitRaw)               // L303 先例
        if (hitFeatures.length > 0) {
            const f = hitFeatures[0] as Feature
            markerCb({ lat: f.getNumberProperty("lat").toDouble(), lng: f.getNumberProperty("lng").toDouble() })
            return true   // 契约不变：{lat,lng} = marker 坐标（properties 直存双精度，零精度损失）
        }
    }
}
```

- **契约**：payload 坐标取自 feature properties（§2 已存），不依赖 geometry 反解；`getNumberProperty` 异常时 fallback `f.geometry() as Point` 的 `latitude()/longitude()`（防御 try/catch）。
- **命中语义变化**（记录）：legacy 添加序优先 + 0.0001 度（≈11m）欧氏容差 → 矩形像素容差 + 渲染层序（label 优先于 marker）；20px 在常规 zoom 档与 11m 容差体验相当。
- **优先级链不变**：P0 聚合 → P1 marker/label → P1.5 图片几何 hit-test → P2 mapClick（L290-366 结构原样）。
- hidden 的 marker/label 不在 source 数据中（§7），天然不命中——替代 legacy "hide 时移出 markerList" 逻辑（L1460-1463）。

## 7. show / hide / remove / clear 改造

| 函数 | 新实现 |
|------|--------|
| `showOverlay(id)` | marker/label：`snapshot.hidden=false` + rebuild 对应共享层（**删除 L1405-1427 的 remove+add 重建路径**）；shape：遍历 `ref.layers` 逐层 `PropertyFactory.visibility(VISIBLE)`（styledPolyline/image 分支不动）；意图先行语义保留：共享层未建（style 未就绪）时只改 snapshot.hidden，ensure 时生效（对齐 v1.9.3 L1398-1401 模式） |
| `hideOverlay(id)` | marker/label：`snapshot.hidden=true` + rebuild（剔除该 feature）；shape：双层 visibility(NONE) |
| `removeOverlay(id)` | marker/label：snapshot 数组 filter + rebuild（共享层保留）；shape：遍历 layers `removeLayer(layer.getId())` + `removeSource(sourceId 字符串)`（L2804 惯例）+ vectorSnapshots filter；image/styledPolyline 分支不动 |
| `clearMarkers()` | 清 marker+label 两个共享层（removeLayer/removeSource 传字符串 id）+ 两 snapshot 数组 + `clearOverlayType('marker')/('label')`（对齐 L578-579）；`markerBitmapRegistered` 不重置（style 未变，位图仍在） |
| `clearPolygons()` | vectorSnapshots 按 `kind=='polygon'` 过滤，逐个摘层摘 source；`clearOverlayType('polygon')` |
| `clearOverlays()` | clearMarkers + clearPolygons + clearImageOverlaysInternal + clearStyledPolylinesInternal（现状）+ 新增 `clearVectorOverlaysInternal()`（polyline/circle/rectangle 全清，含共享 marker/label 层已由 clearMarkers 覆盖） |
| `destroyMap()` | L1100-1104 五列表赋值替换为：三 snapshot 数组清空、marker/labelSourceRef/layerRef=null、`markerBitmapRegistered=false` |
| `showMarkerInfo/hideMarkerInfo` | **no-op + 中文块注释文档化**：MapLibre 11.x native-view 下 InfoWindow 实测不显示（v1.5.6 结论），组件层已有 uvue 自绘气泡（tdt-map.uvue L13-21）；保留函数体与签名，仅 `console.log` 一条诊断日志后 return |

## 8. rebuildUserLayersAfterStyle 扩展（L2575-2641）

在**图片覆盖物重放块之后、geoJsonReplayCallback 之前**插入六类重放块（保持既有 customTile → cluster → heatmap → image 顺序不动）：

```uts
// v1.10.0: 六类注记类覆盖物重放（快照 → 重建 → 恢复可见性，仿 image 三段式 L2602-2634）
// 1) shape 重放：遍历 vectorSnapshots（添加序），addShapeOverlay 内部走锚定层序；
//    visible=false 的建层后立即 visibility(NONE)（"hide 过的不能因重建而复现"，L2622 同款）
// 2) marker 重放：markerBitmapRegistered=false（新 style 位图表已清空，强制重注册）
//    → ensureMarkerStyle(st2) → rebuildMarkerSource()
// 3) label 重放：ensureLabelStyle(st2) 内逐 snapshot 重画位图 + addImage("tdt-label-icon-"+oid)
//    → rebuildLabelSource()（hidden 快照保留，重放不改变）
```

顺序依据：shape 在下、marker/label 在上的固定层序由锚定规则保证（§9），与数组重放顺序解耦；位图重注册时机收敛在 ensure 函数内（幂等，样式切换后首次调用生效）。overlayMap 的 entry 在重放时保持原 id 重建（对齐 L2619 image 模式——shape 条目直接复用，marker/label 条目 nativeRef 本就为 null 无需回填）。

## 9. z-order 规则

层序自底向上固定为：**底图/注记 → customTile(贴底档) → cluster/heatmap → image(按 zIndex) → shape(fill→line) → marker → label →（运行期新增的 geojson 层等 append 层）**

实现（新增锚定 helper，替代裸 addLayer）：

```uts
// 找到第一个存在的锚点层 id；空串 = append 置顶
function overlayAnchorLayerId(style: Style, anchorIds: Array<string>): string
// shape: anchor = ["tdt-marker-layer", "tdt-label-layer"]；同 shape 内 fill 先插、line 后插(line 盖 fill)
// marker: anchor = ["tdt-label-layer"]
// label:  append（恒最顶）
```

- blank/自定义样式无天地图锚点层不影响（锚的是用户层自身相对序）；
- **已知行为差异**（记录）：运行期后续 append 的层（组件层 addGeoJsonLayer 重放、新加图片置顶档）会盖住 marker/label。legacy annotation 是独立 View 层恒置顶，理论上有差异；但 BeaconApp 现网 marker 零调用、geojson 层与 marker 无叠加场景，风险可忽略。如未来需要 marker 恒置顶，可在 addGeoJsonLayer 侧锚定 `tdt-label-layer`（另行任务）。

## 10. 分步实施清单（每步独立可编译，Step 1-6 期间 annotations imports 保留）

| 步骤 | 内容 | 涉改函数/位置 | 预计行数 |
|------|------|--------------|---------|
| 1 | 基础设施（纯增量）：imports 追加 FillLayer/Path/RectF；新类型/常量/状态变量；splitColorAlpha、buildRingJson、buildDefaultPinBitmap、buildLabelBitmap、ensureMarkerStyle、ensureLabelStyle、rebuildMarkerSource、rebuildLabelSource、overlayAnchorLayerId、buildLineSourceLayer、addShapeOverlay | L21-23 imports；L65-92 状态区 | +280 |
| 2 | marker/label 通路：addMarker(L552)/addLabel(L959)/setMarkers(L972) 替换为快照+共享层 | 三个函数体 | ±70 |
| 3 | shape 通路：addPolygon(L583)/addCircle(L904)/addRectangle(L932)/addPolyline(L626) 改调 addShapeOverlay/buildLineSourceLayer；addStyledPolyline(Batch) 内核换 helper（行为零变化） | 五个函数 | ±120 |
| 4 | 管理面：clearMarkers(L570)/clearPolygons(L610)/clearOverlays(L990)/removeOverlay(L1328)/showOverlay(L1395)/hideOverlay(L1448)/destroyMap(L1100)/showMarkerInfo(L2001)/hideMarkerInfo(L2020) | 九个函数 | ±140 |
| 5 | 点击命中：initMap P1 块替换（L316-334） | 一个块 | ±30 |
| 6 | 重放：rebuildUserLayersAfterStyle 插入六类重放块 | 一个函数 | +80 |
| 7 | 清理：删 L18-20 annotations imports、五个 legacy List 及残留引用；package.json 1.9.3→1.10.0；文件头块注释更新 | imports/状态区 | −90 |
| 8 | 验证与交付：demo 回归 → HBuilderX 编译 → 子模块 feature 分支提交 → 主仓 `make fe-sync-uts-map` + `fe-verify-uts-map` → BeaconApp 编译与四组件冒烟 | — | — |

净变化约 +330/−310，文件总量 ~2970 行。分支：`feature/android-annotations-to-style-layers`（从 main 拉），提交拆分按上表步骤（Step1-2 一提交、3 一提交、4 一提交、5+6 一提交、7+8 一提交），Conventional Commits：`feat(android): ...`。

## 11. 验证方案

**A. 插件 demo（example-project，HBuilderX 运行 Android 模拟器/真机）**：
1. 「标记」按钮（L272）：红图钉显示、针尖对准坐标、markertap 回调触发 infoText 更新（坐标=marker 坐标）；
2. 长按添加标记（L232）：同上；
3. 「折线/圆/矩形/多边形」：颜色、半透明填充（`#FF660044` 档 alpha 视觉与迁移前截图对比）、5px 描边；
4. 「文字标注」：显示"天安门"黑字白边文字（legacy 不可见 → 迁移后常显，截图存档）；
5. 移除/隐藏/显示最后一个覆盖物（lastOverlayId 三按钮）：即时生效、隐藏后点击不命中；
6. 「清除覆盖物」：全部消失且无残留层泄漏（切换底图后无复现）；
7. 信息窗口按钮（L511-526）：no-op 不崩溃，日志一条；
8. 底图切换 vec→img→blank→自定义 setStyle：六类覆盖物全部重放、位图重注册、hide 过的不复现、可见性模式保持；
9. setMarkers 批量（如 demo 有批量标记入口）：一次渲染无逐个建层卡顿。

**B. BeaconApp 回归**：
1. 主仓 `make fe-sync-uts-map` → `make fe-verify-uts-map`（同步校验）；
2. skill `verify-uniappx-compile-error` 全量编译通过（Android 平台）；
3. 四现网组件冒烟：radar-echo-map / rainfall-color-map / typhoon-track-map / emergency 相关页——均走 addGeoJsonLayer/imageOverlay/styledPolyline 通路（已确认不触达 annotations 路径），验证边界层、帧图、台风路径渲染正常即无回归。

## 12. 风险清单

| # | 风险 | 缓解 |
|---|------|------|
| 1 | `queryRenderedFeatures(RectF, String...)` 多层 id vararg 在 UTS→Kotlin 编译失败（先例仅单参 L301） | 预案 A：`arrayOf("tdt-label-layer","tdt-marker-layer")` 传参（L294 同款预案）；预案 B：两次单层查询按 label>marker 合并 |
| 2 | `PropertyFactory.iconAnchor("bottom")` / `iconAllowOverlap(true)` / `Style.addImage(String, Bitmap)` 签名与预期不符（11.8.1 API 面） | 列入编译期验证项（文件内 context7 注释惯例）；备选 `Property.ANCHOR_BOTTOM` 常量形态 |
| 3 | symbol icon 渲染是否再除 pixelRatio（style L238 已设 2.0）导致图钉偏大/偏小 | iconSize 常量化一处调整；demo 截图与 legacy 对比定标 |
| 4 | 20px 容差与 legacy 11m 欧氏容差在不同 zoom/密度下体验漂移 | `TAP_TOLERANCE_PX` 常量；验证项 1/2 实测微调 |
| 5 | setStyle 后位图丢失 → marker/label 白块或不渲染 | ensure 函数幂等重注册（§8）；验证项 8 覆盖 blank/自定义两条路径 |
| 6 | label 常显属行为变化（legacy 文字不可见） | 设计已论证为缺陷修正；demo 截图存档；三端语义差异记录在函数注释（Web/iOS 仍交互触发） |
| 7 | `Color.parseColor` 负值位运算（`>>`/`&`）在 UTS/Kotlin Int 语义 | 掩码 `(c>>24)&0xFF` 对负数正确（算术右移后取低 8 位）；验证项 3 半透明档实测 |
| 8 | marker 层被运行期 append 层（geojson 重放等）覆盖 | §9 已知差异文档化；现网零调用；后续可锚定 `tdt-label-layer` 扩展 |
| 9 | 共享层全量 setGeoJson 在超大批量（千级）下的构建开销 | setMarkers 一次 rebuild 已优化；量级远低于聚合先例；超出时后续可按 tile 分片（另行任务，不过度设计） |
| 10 | 重放期 style 短暂为空/未就绪导致 ensure 抛错 | 重放块整体置于 `st2 != null` 守卫内（对齐 L2614-2615）；各 ensure/attach 自带 try/catch + 失败 entry 清理（L2629-2632 模式） |
| 11 | show/hide 意图先行语义在共享层模式下丢失（建层前 hide 被吞） | snapshot.hidden 即意图位，ensure→rebuild 时生效，语义强于旧 nativeRef 判空（L1398 同类问题已根治） |
| 12 | 主仓同步遗漏/脏文件混入 | 遵循 beaconapp-activity-card-creator 同款外科手术提交规则：子模块先提交，主仓仅 `make fe-sync-uts-map` 产物 + `fe-verify-uts-map` 校验后单独提交 |
