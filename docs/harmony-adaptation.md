# beige-tdt-map 鸿蒙 ArkTS 编译适配方案

> 立项：2026-10-04。背景：BeaconApp 打通验证（出鸿蒙包）被 tdt-map 存量 ArkTS 错误系统性阻塞，
> 认证相关文件零错误已通过本轮验证标准。本适配在 uts-map 子模块内完成，DevEco 模拟器联调。
>
> **✅ 结果（2026-10-04 11:31）：850 行 ArkTS 错误 → 0，hvigor BUILD SUCCESSFUL，
> .hap 打包/安装/启动全通（DevEco Pura 90 模拟器），应用前台运行无 TDTMap 错误日志。**

## 0. 实施结果与修订

批次实际执行（错误数递减）：批1 catch 规范化（850→222）→ 批2+3 字面量标注/类型显式化（→164，
中间一次 252 为 hvigor 增量缓存脏数据，清 `.hvigor` 后复测）→ 批4 `map.BaseOverlay` 类型层（→88）
→ 批5 相机 API 重写（→12）→ 批6 收尾（→0，BUILD SUCCESSFUL）。

实施中发现的额外事实（对后续维护重要）：
1. **`Record<K,V>` 类型标注的对象字面量初始化在 hvigor ArkTS 下报 arkts-no-untyped-obj-literals**，
   已改 `Map` + 显式 `set`（TDT_TILE_URLS / TYPE_LAYER_MAP），读取由 `[key]` 改 `.get(key) ?? ''`。
2. **对象字面量仅在"类型标注位置"合法**（`const x: T = {...}`）；`as T` 断言与裸返回字面量会被
   UTS 编译器包装为 UTSJSONObject / 报错。MapKit options 一律先声明显式类型变量再传参。
3. **`ClusterItem` 是 abstract class**（非 interface），需定义子类 `TDTClusterItem extends mapCommon.ClusterItem` 实例化。
4. **华为 MapKit API 勘误**（源码原自造/误记）：`setCameraPosition` → `moveCamera(map.newCameraPosition(pos))`；
   `controller.scrollBy` → `moveCamera(map.scrollBy(x,y))`；`Marker.showInfoWindow/hideInfoWindow`
   → `setInfoWindowVisible(bool)`；fitBounds 可用 `map.newLatLngBounds(bounds, padding)`（本期未用，仍为计算 zoom 方案）。
5. **hvigor 增量缓存会复用陈旧诊断**（行号错位、错误数虚高），修复验证时先删
   `unpackage/dist/dev/app-harmony/.hvigor` 再全量编译。
6. 前端 410 页 vapor 转译约 166s；tdt-map 编译曾长期挡在 entry 前端转译之前（www/import/*.ets 未生成过），
   插件层清零后整链路自然打通。

以下为立项时原文（分类与策略与实际一致，供追溯）。


## 1. 现状盘点

- **错误规模**：hvigor 编译日志（`BeaconApp/unpackage/dist/dev/app-harmony/.hvigor/outputs/build-logs/build.log`）
  共 850 行 ArkTS 错误，**全部**位于 `uni_modules/beige-tdt-map/utssdk/app-harmony/index.ets`（产物 2206 行），
  源头即子模块 `uni_modules/beige-tdt-map/utssdk/app-harmony/index.uts`（2084 行，鸿蒙端专属文件）。
- **错误放大机制**（源码 → 产物的编译行为，已实证）：
  1. 源码 `catch (e: any)` → 产物 `catch (e: Object)` → ArkTS 报 `arkts-no-types-in-catch` + `must be any/unknown` 成对错误；
  2. 源码 `const x: any = {字面量}` → 产物 `new UTSJSONObject({...})` 包装 → 传给 MapKit 具名参数（`LatLng`/`MarkerOptions`…）全部不兼容；
  3. `OverlayEntry.nativeRef: any` → 产物 `Object`：不接受 `null`、无法点访问（`.remove()/.setVisible()`…）。

## 2. 错误分类与修复模式

| # | 错误模式（行数） | 源码根因 | 修复模式 | 源码处数 | 风险 |
|---|---|---|---|---|---|
| A | catch 双报（264）+ `e.message` on Object（~116） | `catch (e: any)`（33 处） | `catch (e)` + `(e as Error).message`；实证：beacon-crypto 鸿蒙端零错误即此写法 | 33 | 低（机械） |
| B | `UTSJSONObject not assignable to LatLng/MarkerOptions/...`（~60） | `const opts: any = {...}` 后传 MapKit API | 显式标注 `mapCommon.MarkerOptions` 等（MapKit 全为 interface，带标注字面量可直接赋值；产物已证明 ArkTS 类型标注可用） | ~30 | 中 |
| C | `Property 'remove/getId/setVisible/...' does not exist on Object`（~150）+ `null not assignable to Object`（60） | `OverlayEntry.nativeRef: any` | `nativeRef: MapOverlayUnion | null`（`map.MapMarker \| map.MapPolyline \| map.MapPolygon \| map.MapCircle \| map.MapOverlay...`），点访问处按 overlay type 字段收窄 + `as` | ~80 | 高（逐处收窄） |
| D | `setCameraPosition does not exist on MapComponentController`（40） | 自造 API（华为 MapKit 无此方法） | `moveCamera(CameraUpdate)`（瞬时）/`animateCamera(update, duration?)`（动画）；工厂：`map.newLatLng(latlng, zoom?)` / `map.newLatLngBounds(bounds, padding?)` / `map.newCameraPosition(cameraPosition)`（已从本机 SDK `@hms.core.map.map.d.ts` 核实） | ~20 调用点 | 高（语义级重写） |
| E | `{base,label}` 字面量类型 + `UTSJSONObject missing base,label`（56）+ untyped obj literals（24+16） | `Record<string, { base: string, label: string }>` 等对象字面量类型作标注 | 提取显式 `type TileLayerIds` / `type OverlayEntry`（type 承接字面量，UTS 规则）；Record → `Map<string, TileLayerIds>` | ~10 | 低 |
| F | 杂项：structural typing（4）、any-unknown（4）、HttpRequestOptions/HuksParam 参数（~18） | 零散写法 | 逐处按同思路修（http 请求 options 显式标注 `http.HttpRequestOptions`） | ~8 | 低 |

## 3. 实施批次（每批一次提交，批后编译验证错误数递减）

1. **批0（基线）**：提交工作区遗留的 OverlayEntry 显式类型化半成品（已完成、两端一致，属 E 类先声）。
2. **批1**：A 类 catch 规范化（33 处）→ 预期消 ~380 行。
3. **批2**：B 类 MapKit options 字面量显式标注。
4. **批3**：E 类对象字面量类型显式化。
5. **批4**：C 类 nativeRef 联合类型 + 点访问收窄（工作量最大，逐函数过）。
6. **批5**：D 类相机 API 重写（moveCamera/newLatLngBounds，含 fitBounds 语义核对）。
7. **批6**：F 类收尾至编译零错误。
8. **联调**：DevEco 模拟器装包冒烟（地图渲染/瓦片/marker/polyline/相机移动/点击回调）。
9. **同步**：子模块提交 → `make fe-sync-uts-map` → BeaconApp 出鸿蒙包复验。

## 4. 关键 API 映射表（本机 SDK 已核实）

| 旧写法（自造/错误） | 新写法（华为 MapKit） |
|---|---|
| `mapController.setCameraPosition({position:{latitude,longitude}, zoom})` | `mapController.moveCamera(map.newLatLng({latitude, longitude} as mapCommon.LatLng, zoom))` |
| 视野适配（fitBounds） | `mapController.moveCamera(map.newLatLngBounds(bounds, padding))` |
| 带动画相机 | `mapController.animateCamera(update, duration?)` |
| `catch (e: any)` + `e.message` | `catch (e)` + `(e as Error).message` |
| `const opts: any = {...}` 传 MapKit | `const opts: mapCommon.MarkerOptions = {...}` |

## 5. 风险与回退

- 全部改动限定在 `app-harmony/index.uts`（+必要时 `builder.ets`），不影响 Android/iOS/Web 三端。
- C 类收窄若逐处 `as` 导致运行时类型不匹配，以 overlay `type` 字段判断保证分支正确性。
- D 类语义：`moveCamera` 为无动画瞬时设置，与原 `setCameraPosition` 意图一致；fitBounds 的 padding 语义需在模拟器上目视核对。
- 每批独立提交，可单独回退。
