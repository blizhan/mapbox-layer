# OM data_spatial, tiles, ranges, and picker flow

本说明文档基于仓库当前实现，集中解释以下问题：

1. `data_spatial` 数据如何与 zoom/tile 坐标关联
2. `ranges` 如何计算与使用
3. picker（点查询）如何从 OM 数据中取值并返回数值
4. 不同网格类型（regular / projected / gaussian）在上述流程里的角色

> 说明：本仓库的实现中 **OM 文件不按 zoom 存多份数据**。zoom 与 `{z}/{x}/{y}` 只用于瓦片坐标与经纬度转换。

---

## 1. 总览：从 URL 到瓦片渲染的主流程

```
om://.../file.om?variable=xxx/{z}/{x}/{y}
        │
        ▼
parseUrlComponents (解析 tileIndex)
        │
        ▼
parseRequest
  ├─ resolve domain / variable
  └─ dataOptions.bounds = currentBounds
        │
        ▼
getOrCreateState
  ├─ grid.getCoveringRanges(bounds) => ranges
  └─ OMapsFileReader.readVariable(variable, ranges)
        │
        ▼
worker: tile loop
  ├─ tile2lat / tile2lon
  └─ grid.getLinearInterpolatedValue(values, lat, lon)
        │
        ▼
Image/Vector tile output
```

**核心结论**：
- `ranges` 来自 **地图视窗 bounds**，而非 `{z}/{x}/{y}`。
- tile 坐标只负责 **瓦片像素 → 经纬度** 的映射。

---

## 2. data_spatial 与 zoom 的关系

### 2.1 OM 文件不是多 zoom 数据
- OM 文件可以包含多个变量，但**每个变量**只对应一份网格数据（Float32Array），不包含多级 zoom 金字塔。
- zoom 并不改变数据精度或存储层次，只影响渲染采样密度。

### 2.2 tile 坐标的作用
- `{z}/{x}/{y}` 通过 `tile2lat/tile2lon` 转成经纬度。
- 对应代码逻辑：

```
lat = tile2lat(y + i / tileSize, z)
lon = tile2lon(x + j / tileSize, z)
```

因此，zoom 只是决定 “瓦片像素点落在哪个经纬度位置”。

---

## 3. ranges 的计算逻辑

### 3.1 bounds 来源
- `currentBounds` 由 `updateCurrentBounds` 根据地图视窗计算。
- 该步骤使用 `tilebelt` 将视窗对齐到瓦片 BBOX。

### 3.2 ranges 生成入口
- `getOrCreateState` 中：
  - 如果有 `dataOptions.bounds`，就调用 `grid.getCoveringRanges(...)`。
  - 否则读取全量网格。

### 3.3 RegularGrid 的 ranges 算法（概念）
1. 将 south/west/north/east 对齐到网格步长 `dx/dy`。
2. 用 `(coord - origin) / d` 转换到网格索引。
3. 通过 floor/ceil 生成 `[min, max]`。
4. 返回 `[{start: minY, end: maxY}, {start: minX, end: maxX}]`。

**注意**：ranges 的目的是裁剪 OM 文件读取范围，以减少数据量。

---

## 4. Picker（点查询）取值流程

### 4.1 入口
- `getValueFromLatLong(lat, lon, omUrl)`

### 4.2 执行步骤
1. 从 URL 解析出 `fileAndVariableKey`，找到已缓存的 state。
2. 使用 state 的 `ranges` 创建 grid。
3. `grid.getLinearInterpolatedValue(values, lat, lon)` 返回数值。

### 4.3 数值化发生的位置
- OM 数据读出就是 `Float32Array`。
- 对 UV 风等变量会在读取时派生 speed/direction。
- picker 只做 **插值查询**，不做额外数值转换。

---

## 5. 不同网格类型的影响

### 5.1 RegularGrid
- 直接在经纬度空间中进行索引计算。
- `lat/lon` => `(x,y)` => 双线性插值。

### 5.2 ProjectionGrid
- 先通过投影把经纬度转换到本地投影坐标。
- 再根据 `dx/dy` 和 origin 计算索引与插值。

### 5.3 GaussianGrid
- 使用高斯网格的特殊 y 索引计算规则。
- 但整体流程仍是：经纬度 ➜ 索引 ➜ 插值。

---

## 6. 常见疑问答疑

### Q1: zoom 是否影响读取 OM 数据的范围？
**不会**。`ranges` 来自视窗 bounds；tile 坐标仅决定像素级采样位置。

### Q2: picker 是按 tile 返回的颜色值还是原始数值？
**原始数值**。picker 直接返回插值后的浮点值。

### Q3: 为什么 tile 需要 worker 读取 OM 数据？
- worker 负责对每个瓦片像素进行插值、着色、绘制。
- 这样主线程不会被大规模计算阻塞。

---

## 7. 关键函数索引（便于跳转）

- URL / tileIndex 解析：`src/utils/parse-url.ts`
- request 解析与 bounds：`src/utils/parse-request.ts`
- bounds 更新：`src/utils/bounds.ts`
- ranges 计算 & state：`src/om-protocol-state.ts`
- tile ➜ 经纬度：`src/utils/math.ts`
- worker 渲染：`src/worker.ts`
- grid 实现：`src/grids/*`
- OM 读取 & 派生变量：`src/om-file-reader.ts`
- picker 值查询：`getValueFromLatLong` in `src/om-protocol-state.ts`
