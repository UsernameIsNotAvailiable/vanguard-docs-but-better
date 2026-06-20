# Math

`Vanguard.Math` and `Vanguard.Util.Math` provide number helpers commonly needed
for gameplay interpolation, remapping, looping, easing, quantization, and
floating-point comparison.

```lua
local Math = Vanguard.Math
```

All helpers validate their numeric contracts. Invalid values use
[`VG-MATH-001`](../../errors/index.md#vg-math-001); invalid ranges use
[`VG-MATH-002`](../../errors/index.md#vg-math-002).

## Constants

| Constant | Value | Purpose |
| --- | ---: | --- |
| `EPSILON` | `1e-6` | Default relative tolerance for approximate equality |
| `TAU` | `math.pi * 2` | One full turn in radians |

## Rounding And Quantization

### round

```lua
Math.round(12.345, 2) -- 12.35
Math.round(12.345) -- 12
```

`decimalPlaces` must be an integer and may be negative to round to tens,
hundreds, and larger places.

### snap

```lua
Math.snap(12, 5) -- 10
Math.snap(13, 5) -- 15
```

The increment must be greater than zero.

## Interpolation And Mapping

### lerp

```lua
Math.lerp(10, 20, 0.25) -- 12.5
```

`alpha` is not clamped, so values below zero or above one extrapolate.

### inverseLerp

```lua
Math.inverseLerp(10, 20, 12.5) -- 0.25
```

The end value must be greater than the start value. The result is not clamped.

### map

```lua
Math.map(5, 0, 10, 0, 100) -- 50
Math.map(20, 0, 10, 0, 100, true) -- 100
```

The optional final boolean clamps normalized progress before mapping it to the
output range. Output ranges may ascend or descend.

## Movement And Easing

### moveTowards

```lua
speed = Math.moveTowards(speed, targetSpeed, acceleration * deltaTime)
```

Moves by at most `maxDelta` without overshooting. `maxDelta` must be zero or
greater.

### smoothstep

```lua
local alpha = Math.smoothstep(0, 10, distance)
```

Normalizes and clamps the value, then applies cubic smoothing.

### smootherstep

```lua
local alpha = Math.smootherstep(0, 10, distance)
```

Uses a fifth-order curve with zero first and second derivatives at both ends.

## Repeating Values

### wrap

```lua
Math.wrap(12, 0, 10) -- 2
Math.wrap(-1, 0, 10) -- 9
```

Returns a value in `[minimum, maximum)`. Maximum must be greater than minimum.

### pingPong

```lua
Math.pingPong(elapsed, 5)
```

Repeats between zero and `length`, reflecting at each endpoint. Length must be
greater than zero.

## Comparison And Aggregation

### approximatelyEqual

```lua
Math.approximatelyEqual(0.1 + 0.2, 0.3) -- true
Math.approximatelyEqual(a, b, 1e-4)
```

Comparison is relative to the largest magnitude or one, avoiding a tolerance
that becomes useless for large values. Epsilon must be non-negative.

### average

```lua
Math.average(2, 4, 6) -- 4
```

Requires at least one number.

## API Summary

| Function | Signature |
| --- | --- |
| `round` | `(value, decimalPlaces?) -> number` |
| `lerp` | `(startValue, endValue, alpha) -> number` |
| `inverseLerp` | `(startValue, endValue, value) -> number` |
| `map` | `(value, inMin, inMax, outMin, outMax, clampResult?) -> number` |
| `approximatelyEqual` | `(left, right, epsilon?) -> boolean` |
| `wrap` | `(value, minimum, maximum) -> number` |
| `pingPong` | `(value, length) -> number` |
| `smoothstep` | `(minimum, maximum, value) -> number` |
| `smootherstep` | `(minimum, maximum, value) -> number` |
| `moveTowards` | `(current, target, maxDelta) -> number` |
| `snap` | `(value, increment) -> number` |
| `average` | `(...values) -> number` |
