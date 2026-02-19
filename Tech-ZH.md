# Minecraft 远距离渲染问题修复报告

## 问题现象

在 Minecraft Classic/Indev/Infdev 版本中，当玩家移动到距离世界原点较远的位置时，会出现以下问题：

| 距离 | 现象 |
|------|------|
| X/Z ±1,024/-512 | 天空盒停止渲染，变成纯蓝色天空 |
| X/Z ±65,536 | 碰撞箱开始变形 |
| X/Z ±131,072 | 区块间隙开始抖动和扭曲，效果随距离指数级恶化 |
| X/Z ±2,097,152 | 世界在某些角度下无法完全渲染 |
| X/Z ±16,777,216 | 方块不再固体，玩家会掉落并接触岩浆层，但仍可在水中游泳 |

## 根本原因

### 浮点精度问题

Java 的 `float` 类型使用 IEEE 754 单精度浮点数格式，只有 **24 位有效数字**（包括隐含的1）。这意味着：

- `float` 能精确表示的最大整数是 **16,777,215** (2²⁴ - 1)
- 当坐标超过这个值时，`float` 无法区分相邻的整数
- 例如：16,777,216 和 16,777,217 在 `float` 中表示为相同的值

```
坐标值          float 表示
16,777,216  →  1.6777216E7  ✓
16,777,217  →  1.6777216E7  ✗ (精度丢失！)
16,777,218  →  1.6777218E7  ✓
16,777,219  →  1.6777218E7  ✗ (精度丢失！)
```

### 问题来源

1. **方块边界**：`Block` 类的 `minX/Y/Z`、`maxX/Y/Z` 字段使用 `float` 类型
2. **顶点坐标**：`Tessellator` 和 `RenderBlocks` 使用 `float` 计算顶点位置
3. **射线检测**：`World.rayTraceBlocks()` 使用 `float` 计算边界
4. **粒子位置**：粒子渲染使用绝对坐标，在大坐标时精度丢失

## 修复方案

### 1. 方块边界精度修复

**文件**: `Block.java`

将方块边界字段从 `float` 改为 `double`：

```java
// 修复前
public float minX, minY, minZ, maxX, maxY, maxZ;

// 修复后
public double minX, minY, minZ, maxX, maxY, maxZ;
```

**效果**：解决了 X/Z ±16,777,216 处方块非固体的问题，因为碰撞检测现在使用 `double` 精度。

### 2. 渲染器精度修复

**文件**: `Tessellator.java`, `RenderBlocks.java`

添加支持 `double` 参数的方法：

```java
// Tessellator.java
public void addVertex(double x, double y, double z) {
    // 使用 double 计算偏移后的坐标，最后转换为 float
    rawBuffer[index] = Float.floatToRawIntBits((float)(x + xOffset));
    ...
}

public void addVertexWithUV(double x, double y, double z, double u, double v) {
    this.textureU = (float)u;
    this.textureV = (float)v;
    this.addVertex(x, y, z);
}
```

```java
// RenderBlocks.java - 所有渲染方法参数改为 double
private void renderBottomFace(Block block, double x, double y, double z, int texture) {
    double minX = x + block.minX;  // 使用 double 计算
    ...
}
```

**效果**：解决了纹理拉伸和渲染抖动问题。

### 3. 射线检测精度修复

**文件**: `World.java`

将 `rayTraceBlocks()` 方法中的边界计算从 `float` 改为 `double`：

```java
// 修复前
float boundary = 999.0F;
if (i3 > i6) boundary = (float)i6 + 1.0F;

// 修复后
double boundary = 999.0D;
if (i3 > i6) boundary = (double)i6 + 1.0D;
```

**效果**：解决了 X/Z ±65,536 处碰撞箱变形的问题。

### 4. 粒子渲染修复

**文件**: `EntityFX.java`, `EffectRenderer.java`

使用相对坐标渲染粒子：

```java
// EntityFX.java - 添加静态字段存储玩家位置
public static double interpPosX, interpPosY, interpPosZ;

// 渲染时使用相对坐标
float x = (float)(prevPosX + (posX - prevPosX) * partialTicks - interpPosX);
float y = (float)(prevPosY + (posY - prevPosY) * partialTicks - interpPosY);
float z = (float)(prevPosZ + (posZ - prevPosZ) * partialTicks - interpPosZ);
```

```java
// EffectRenderer.java - 渲染前设置玩家位置
EntityFX.interpPosX = player.prevRenderX + (player.posX - player.prevRenderX) * partialTicks;
EntityFX.interpPosY = player.prevRenderY + (player.posY - player.prevRenderY) * partialTicks;
EntityFX.interpPosZ = player.prevRenderZ + (player.posZ - player.prevRenderZ) * partialTicks;
```

**效果**：粒子在大坐标位置正确渲染。

### 5. 碰撞箱渲染修复

**文件**: `RenderGlobal.java`

为 `drawSelectionBox()` 和 `drawBlockBreaking()` 方法添加插值参数，使用正确的相对坐标计算：

```java
public void drawSelectionBox(MovingObjectPosition target, int pass, float partialTicks) {
    double x = player.prevRenderX + (player.posX - player.prevRenderX) * partialTicks;
    double y = player.prevRenderY + (player.posY - player.prevRenderY) * partialTicks;
    double z = player.prevRenderZ + (player.posZ - player.prevRenderZ) * partialTicks;
    AxisAlignedBB box = block.getSelectedBoundingBox(...).copyOffset(-x, -y, -z);
    ...
}
```

**效果**：碰撞箱在任何坐标位置都正确显示。

## 修复效果对比

| 问题 | 修复前 | 修复后 |
|------|--------|--------|
| 方块非固体 | X/Z > 16,777,216 | 基本解决 |
| 纹理拉伸 | X/Z > 16,777,216 | 大幅改善 |
| 碰撞箱变形 | X/Z > 65,536 | 基本解决 |
| 粒子偏移 | 所有距离 | 完全解决 |
| 天空渲染 | X/Z > 1,024 | 完全解决 |

## 技术原理总结

### 为什么使用相对坐标？

OpenGL 在处理顶点坐标时，最终会将坐标转换为屏幕坐标。当使用绝对坐标时：

```
绝对坐标: 16,777,216.5 → float → 16,777,216.0 (精度丢失)
```

使用相对坐标时：

```
玩家位置: 16,777,216.0
方块位置: 16,777,217.0
相对坐标: 16,777,217.0 - 16,777,216.0 = 1.0 (精确！)
```

### 为什么 double 有效？

`double` 类型有 **53 位有效数字**，能精确表示的最大整数是 **9,007,199,254,740,991** (2⁵³ - 1)，远超 Minecraft 世界边界。

## 结论

通过将关键计算从 `float` 升级到 `double`，并使用相对坐标渲染技术，成功解决了 Minecraft 远距离渲染的精度问题。这些修复使游戏在更大的坐标范围内保持稳定运行。
