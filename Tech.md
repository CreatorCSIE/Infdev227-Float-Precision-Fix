# Minecraft 远距离渲染修复总结
# Minecraft Long Distance Rendering Fix Summary
# 由GLM-5总结
# Summarized by GLM-5

---

## 中文版

### 问题背景

在 Minecraft 原版中，当玩家坐标超过 16777216（2^24）时，会出现以下问题：
- 方块变为非固体
- 纹理被拉伸
- 水渲染消失
- 碰撞箱显示错误
- 粒子位置偏移
- 太阳/月亮跟随玩家移动

这些问题的根本原因是 `float` 类型的精度限制。`float` 只有 23 位尾数，最大精确表示 2^24 = 16777216，超过这个值就会丢失精度。

### 修复内容

#### 1. Entity 类 - 添加渲染插值位置字段

**文件**: `net/minecraft/game/entity/Entity.java`

在类末尾添加了三个新字段，用于存储玩家的渲染插值位置：
- `prevRenderX` - 上一帧渲染时的 X 坐标
- `prevRenderY` - 上一帧渲染时的 Y 坐标
- `prevRenderZ` - 上一帧渲染时的 Z 坐标

这些字段在每帧开始时更新，用于计算相对坐标。

#### 2. Block 类 - 边界字段精度提升

**文件**: `net/minecraft/game/world/block/Block.java`

将方块边界字段从 `float` 改为 `double`：
- `minX`, `minY`, `minZ`
- `maxX`, `maxY`, `maxZ`

同时更新了相关方法：
- `setBlockBounds()` - 使用 double 类型参数
- `getSelectedBoundingBoxFromPool()` - 返回精确的碰撞箱
- `getCollisionBoundingBoxFromPool()` - 返回精确的碰撞箱

#### 3. Tessellator 类 - 添加 double 类型支持

**文件**: `net/minecraft/client/render/Tessellator.java`

- 添加了 `addVertexWithUV(double, double, double, double, double)` 方法
- 添加了 `addVertex(double, double, double)` 方法
- 将偏移量字段 `xOffset`, `yOffset`, `zOffset` 改为 `double` 类型

#### 4. RenderBlocks 类 - 渲染方法精度提升

**文件**: `net/minecraft/client/render/RenderBlocks.java`

将所有渲染方法的参数从 `float` 改为 `double`：
- `renderBottomFace()`
- `renderTopFace()`
- `renderEastFace()`
- `renderWestFace()`
- `renderNorthFace()`
- `renderSouthFace()`
- `renderCrossedSquares()`
- `renderBlockCropsImpl()`

移除了不再需要的 `posX`, `posY`, `posZ` 字段。

#### 5. RenderGlobal 类 - 透明方块渲染和天空修复

**文件**: `net/minecraft/client/render/RenderGlobal.java`

- 添加了 `renderList` 列表存储需要渲染的透明方块
- 修改 `renderSortedRenderers()` 方法检查 `skipRenderPass`
- 添加 `renderRenderList()` 方法执行实际渲染
- 添加 `renderAllRenderLists(int, double)` 方法
- 修复 `renderSky()` 方法，移除太阳/月亮的位置偏移
- 修复 `drawSelectionBox()` 和 `drawBlockBreaking()` 方法，添加插值参数

#### 6. WorldRenderer 类 - 访问权限修改

**文件**: `net/minecraft/client/render/WorldRenderer.java`

将 `skipRenderPass` 字段从 `private` 改为 `public`，允许 RenderGlobal 访问。

#### 7. EntityRenderer 类 - 渲染调用更新

**文件**: `net/minecraft/client/render/EntityRenderer.java`

- 更新透明方块渲染流程，正确绑定纹理
- 更新 `drawBlockBreaking` 和 `drawSelectionBox` 调用，传递插值参数

#### 8. World 类 - 射线检测精度提升

**文件**: `net/minecraft/game/world/World.java`

将 `rayTraceBlocks()` 方法中的边界值从 `float` 改为 `double`，修复大坐标下的碰撞检测问题。

#### 9. EntityFX 类 - 粒子渲染修复

**文件**: `net/minecraft/client/effect/EntityFX.java`

添加静态字段存储玩家插值位置：
- `interpPosX`
- `interpPosY`
- `interpPosZ`

修改 `renderParticle()` 方法使用相对坐标计算粒子位置。

#### 10. EffectRenderer 类 - 粒子管理修复

**文件**: `net/minecraft/client/effect/EffectRenderer.java`

- 在 `renderParticles()` 方法中设置 `EntityFX.interpPosX/Y/Z`
- 将 `addBlockDestroyEffects()` 和 `addBlockHitEffects()` 方法中的坐标计算改为 `double`

#### 11. EntityDiggingFX 类 - 挖掘粒子修复

**文件**: `net/minecraft/client/effect/EntityDiggingFX.java`

- 将构造函数参数从 `float` 改为 `double`
- 修改 `renderParticle()` 方法使用相对坐标

#### 12. 其他文件修复

- `Render.java` - 修复方块阴影渲染的类型转换
- `EffectRenderer.java` (addBlockHitEffects) - 修复方块边界访问的类型转换

### 修复原理

所有修复的核心原理是：**在计算过程中保持 double 精度，只在最后一步转换为 float**。

具体策略：
1. **相对坐标系统**：使用 Tessellator 的 `setTranslation()` 设置偏移量，顶点坐标使用相对值
2. **延迟类型转换**：所有坐标计算使用 double，只在最终渲染时转换
3. **插值位置**：粒子等效果使用相对玩家位置的坐标计算

---

## English Version

### Background

In the original Minecraft, when player coordinates exceed 16777216 (2^24), the following issues occur:
- Blocks become non-solid
- Textures are stretched
- Water rendering disappears
- Bounding box display is incorrect
- Particle positions are offset
- Sun/moon follows the player

The root cause of these issues is the precision limitation of the `float` type. `float` has only 23 bits of mantissa, which can precisely represent up to 2^24 = 16777216. Values beyond this lose precision.

### Fixes

#### 1. Entity Class - Add Render Interpolation Position Fields

**File**: `net/minecraft/game/entity/Entity.java`

Added three new fields at the end of the class to store player render interpolation positions:
- `prevRenderX` - X coordinate at previous frame render
- `prevRenderY` - Y coordinate at previous frame render
- `prevRenderZ` - Z coordinate at previous frame render

These fields are updated at the start of each frame for calculating relative coordinates.

#### 2. Block Class - Boundary Field Precision Upgrade

**File**: `net/minecraft/game/world/block/Block.java`

Changed block boundary fields from `float` to `double`:
- `minX`, `minY`, `minZ`
- `maxX`, `maxY`, `maxZ`

Also updated related methods:
- `setBlockBounds()` - Use double type parameters
- `getSelectedBoundingBoxFromPool()` - Return precise bounding box
- `getCollisionBoundingBoxFromPool()` - Return precise bounding box

#### 3. Tessellator Class - Add Double Type Support

**File**: `net/minecraft/client/render/Tessellator.java`

- Added `addVertexWithUV(double, double, double, double, double)` method
- Added `addVertex(double, double, double)` method
- Changed offset fields `xOffset`, `yOffset`, `zOffset` to `double` type

#### 4. RenderBlocks Class - Rendering Method Precision Upgrade

**File**: `net/minecraft/client/render/RenderBlocks.java`

Changed all rendering method parameters from `float` to `double`:
- `renderBottomFace()`
- `renderTopFace()`
- `renderEastFace()`
- `renderWestFace()`
- `renderNorthFace()`
- `renderSouthFace()`
- `renderCrossedSquares()`
- `renderBlockCropsImpl()`

Removed the no longer needed `posX`, `posY`, `posZ` fields.

#### 5. RenderGlobal Class - Transparent Block Rendering and Sky Fix

**File**: `net/minecraft/client/render/RenderGlobal.java`

- Added `renderList` to store transparent blocks for rendering
- Modified `renderSortedRenderers()` method to check `skipRenderPass`
- Added `renderRenderList()` method for actual rendering
- Added `renderAllRenderLists(int, double)` method
- Fixed `renderSky()` method, removed sun/moon position offset
- Fixed `drawSelectionBox()` and `drawBlockBreaking()` methods, added interpolation parameter

#### 6. WorldRenderer Class - Access Permission Modification

**File**: `net/minecraft/client/render/WorldRenderer.java`

Changed `skipRenderPass` field from `private` to `public` to allow RenderGlobal access.

#### 7. EntityRenderer Class - Rendering Call Updates

**File**: `net/minecraft/client/render/EntityRenderer.java`

- Updated transparent block rendering flow, correctly bind textures
- Updated `drawBlockBreaking` and `drawSelectionBox` calls, passing interpolation parameter

#### 8. World Class - Ray Tracing Precision Upgrade

**File**: `net/minecraft/game/world/World.java`

Changed boundary values in `rayTraceBlocks()` method from `float` to `double`, fixing collision detection issues at large coordinates.

#### 9. EntityFX Class - Particle Rendering Fix

**File**: `net/minecraft/client/effect/EntityFX.java`

Added static fields to store player interpolation position:
- `interpPosX`
- `interpPosY`
- `interpPosZ`

Modified `renderParticle()` method to use relative coordinates for particle position calculation.

#### 10. EffectRenderer Class - Particle Management Fix

**File**: `net/minecraft/client/effect/EffectRenderer.java`

- Set `EntityFX.interpPosX/Y/Z` in `renderParticles()` method
- Changed coordinate calculation in `addBlockDestroyEffects()` and `addBlockHitEffects()` methods to `double`

#### 11. EntityDiggingFX Class - Digging Particle Fix

**File**: `net/minecraft/client/effect/EntityDiggingFX.java`

- Changed constructor parameters from `float` to `double`
- Modified `renderParticle()` method to use relative coordinates

#### 12. Other File Fixes

- `Render.java` - Fixed type conversion for block shadow rendering
- `EffectRenderer.java` (addBlockHitEffects) - Fixed type conversion for block boundary access

### Fix Principle

The core principle of all fixes is: **Maintain double precision during calculations, only convert to float at the final step**.

Specific strategies:
1. **Relative Coordinate System**: Use Tessellator's `setTranslation()` to set offset, use relative values for vertex coordinates
2. **Delayed Type Conversion**: All coordinate calculations use double, only convert at final rendering
3. **Interpolation Position**: Effects like particles use coordinates relative to player position

---

## 修改文件列表 / Modified Files List

| 文件路径 / File Path | 修改类型 / Modification Type |
|---------------------|------------------------------|
| `net/minecraft/game/entity/Entity.java` | 添加字段 / Added fields |
| `net/minecraft/game/world/block/Block.java` | 字段类型修改 / Field type change |
| `net/minecraft/client/render/Tessellator.java` | 添加方法 / Added methods |
| `net/minecraft/client/render/RenderBlocks.java` | 参数类型修改 / Parameter type change |
| `net/minecraft/client/render/RenderGlobal.java` | 多处修改 / Multiple changes |
| `net/minecraft/client/render/WorldRenderer.java` | 访问权限修改 / Access permission change |
| `net/minecraft/client/render/EntityRenderer.java` | 方法调用更新 / Method call updates |
| `net/minecraft/game/world/World.java` | 变量类型修改 / Variable type change |
| `net/minecraft/client/effect/EntityFX.java` | 添加字段、修改方法 / Added fields, modified method |
| `net/minecraft/client/effect/EffectRenderer.java` | 多处修改 / Multiple changes |
| `net/minecraft/client/effect/EntityDiggingFX.java` | 参数类型修改 / Parameter type change |
| `net/minecraft/client/render/entity/Render.java` | 类型转换修复 / Type conversion fix |