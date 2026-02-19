# Minecraft Far Lands Rendering Fix Report

## Problem Description

In Minecraft Classic/Indev/Infdev versions, when the player moves far from the world origin, the following issues occur:

| Distance | Phenomenon |
|----------|------------|
| X/Z ±1,024/-512 | The skybox around the world stops rendering, giving way to a blank light blue sky |
| X/Z ±65,536 | The hitbox begins to lose its shape |
| X/Z ±131,072 | The gaps between chunks begin to become jumpy and distorted. These effects double every exponent of two |
| X/Z ±2,097,152 | The effects become so severe that the world seems to not render completely at certain angles |
| X/Z ±16,777,216 | Blocks are no longer solid, allowing the player to fall and hit a layer of lava, however the player can still swim through water |

## Root Cause

### Floating-Point Precision Issues

Java's `float` type uses IEEE 754 single-precision floating-point format with only **24 bits of significand** (including the implicit 1). This means:

- The maximum integer that `float` can precisely represent is **16,777,215** (2²⁴ - 1)
- When coordinates exceed this value, `float` cannot distinguish between adjacent integers
- For example: 16,777,216 and 16,777,217 are represented as the same value in `float`

```
Coordinate      float representation
16,777,216  →  1.6777216E7  ✓
16,777,217  →  1.6777216E7  ✗ (precision lost!)
16,777,218  →  1.6777218E7  ✓
16,777,219  →  1.6777218E7  ✗ (precision lost!)
```

### Sources of Problems

1. **Block Boundaries**: `Block` class fields `minX/Y/Z`, `maxX/Y/Z` use `float` type
2. **Vertex Coordinates**: `Tessellator` and `RenderBlocks` use `float` to calculate vertex positions
3. **Ray Tracing**: `World.rayTraceBlocks()` uses `float` to calculate boundaries
4. **Particle Positions**: Particle rendering uses absolute coordinates, losing precision at large coordinates

## Fix Solutions

### 1. Block Boundary Precision Fix

**File**: `Block.java`

Changed block boundary fields from `float` to `double`:

```java
// Before fix
public float minX, minY, minZ, maxX, maxY, maxZ;

// After fix
public double minX, minY, minZ, maxX, maxY, maxZ;
```

**Effect**: Resolved the non-solid blocks issue at X/Z ±16,777,216, as collision detection now uses `double` precision.

### 2. Renderer Precision Fix

**Files**: `Tessellator.java`, `RenderBlocks.java`

Added methods supporting `double` parameters:

```java
// Tessellator.java
public void addVertex(double x, double y, double z) {
    // Use double to calculate offset coordinates, then convert to float
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
// RenderBlocks.java - All render methods changed to use double parameters
private void renderBottomFace(Block block, double x, double y, double z, int texture) {
    double minX = x + block.minX;  // Use double for calculation
    ...
}
```

**Effect**: Resolved texture stretching and rendering jitter issues.

### 3. Ray Tracing Precision Fix

**File**: `World.java`

Changed boundary calculations in `rayTraceBlocks()` from `float` to `double`:

```java
// Before fix
float boundary = 999.0F;
if (i3 > i6) boundary = (float)i6 + 1.0F;

// After fix
double boundary = 999.0D;
if (i3 > i6) boundary = (double)i6 + 1.0D;
```

**Effect**: Resolved hitbox deformation at X/Z ±65,536.

### 4. Particle Rendering Fix

**Files**: `EntityFX.java`, `EffectRenderer.java`

Use relative coordinates for particle rendering:

```java
// EntityFX.java - Add static fields to store player position
public static double interpPosX, interpPosY, interpPosZ;

// Use relative coordinates during rendering
float x = (float)(prevPosX + (posX - prevPosX) * partialTicks - interpPosX);
float y = (float)(prevPosY + (posY - prevPosY) * partialTicks - interpPosY);
float z = (float)(prevPosZ + (posZ - prevPosZ) * partialTicks - interpPosZ);
```

```java
// EffectRenderer.java - Set player position before rendering
EntityFX.interpPosX = player.prevRenderX + (player.posX - player.prevRenderX) * partialTicks;
EntityFX.interpPosY = player.prevRenderY + (player.posY - player.prevRenderY) * partialTicks;
EntityFX.interpPosZ = player.prevRenderZ + (player.posZ - player.prevRenderZ) * partialTicks;
```

**Effect**: Particles render correctly at large coordinates.

### 5. Sky and Cloud Rendering Fix

**File**: `RenderGlobal.java`

- Removed player position offset from sky rendering, making sun/moon render relative to world origin
- Cloud height dynamically calculated based on player Y coordinate

**Effect**: Sky and clouds render correctly at any coordinate position.

### 6. Selection Box Rendering Fix

**File**: `RenderGlobal.java`

Added interpolation parameters to `drawSelectionBox()` and `drawBlockBreaking()` methods, using correct relative coordinate calculation:

```java
public void drawSelectionBox(MovingObjectPosition target, int pass, float partialTicks) {
    double x = player.prevRenderX + (player.posX - player.prevRenderX) * partialTicks;
    double y = player.prevRenderY + (player.posY - player.prevRenderY) * partialTicks;
    double z = player.prevRenderZ + (player.posZ - player.prevRenderZ) * partialTicks;
    AxisAlignedBB box = block.getSelectedBoundingBox(...).copyOffset(-x, -y, -z);
    ...
}
```

**Effect**: Selection box displays correctly at any coordinate position.

## Fix Results Comparison

| Issue | Before Fix | After Fix |
|-------|------------|-----------|
| Non-solid blocks | X/Z > 16,777,216 | Mostly resolved |
| Texture stretching | X/Z > 16,777,216 | Significantly improved |
| Hitbox deformation | X/Z > 65,536 | Mostly resolved |
| Particle offset | All distances | Fully resolved |
| Sky rendering | X/Z > 1,024 | Fully resolved |
| Clouds following player | All distances | Fully resolved |

## Technical Principles Summary

### Why Use Relative Coordinates?

When OpenGL processes vertex coordinates, it ultimately converts them to screen coordinates. When using absolute coordinates:

```
Absolute coordinate: 16,777,216.5 → float → 16,777,216.0 (precision lost)
```

When using relative coordinates:

```
Player position: 16,777,216.0
Block position:  16,777,217.0
Relative coordinate: 16,777,217.0 - 16,777,216.0 = 1.0 (precise!)
```

### Why Does Double Work?

The `double` type has **53 bits of significand**, capable of precisely representing integers up to **9,007,199,254,740,991** (2⁵³ - 1), far exceeding the Minecraft world boundary.

## Conclusion

By upgrading key calculations from `float` to `double` and using relative coordinate rendering techniques, the precision issues in Minecraft's far distance rendering have been successfully resolved. These fixes enable the game to maintain stable operation across a much larger coordinate range.
