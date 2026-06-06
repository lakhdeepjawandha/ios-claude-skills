# Graphics Performance Reference

## Instruments — where to start

1. **Core Animation instrument** — FPS, commit cost, offscreen rendering passes (shown in yellow)
2. **GPU Frame Capture** (Xcode → Debug bar → camera icon) — shader execution time, draw call count, texture memory
3. **Allocations instrument** — detect texture memory leaks
4. **Time Profiler** — find CPU bottlenecks in your draw code

Key metrics:
- Target: 60fps = 16.7ms per frame, 120fps (ProMotion) = 8.3ms
- Offscreen rendering = red overlay in Simulator → Debug → Color Offscreen-Rendered

---

## Offscreen rendering — avoid unless necessary

Offscreen rendering means the GPU must switch to a new render pass to composite:

**Triggers** (avoid these on frequently-animated layers):
- `layer.shouldRasterize = true` (only use for static, complex layers)
- `layer.mask` (use `layer.cornerRadius` with `layer.masksToBounds` instead when possible)
- `layer.shadowPath` not set (always set `shadowPath` manually for custom shapes)
- Any layer with `allowsGroupOpacity` + `opacity < 1`
- `UIVisualEffectView` (intentional — OK, but know the cost)

**Solutions**:
```swift
// BAD — offscreen rendering for shadow
layer.shadowOffset = CGSize(width: 0, height: 4)
layer.shadowRadius = 8
layer.shadowOpacity = 0.3

// GOOD — provide shadowPath to skip offscreen pass
layer.shadowPath = UIBezierPath(roundedRect: layer.bounds, cornerRadius: 12).cgPath
layer.shadowOffset = CGSize(width: 0, height: 4)
layer.shadowRadius = 8
layer.shadowOpacity = 0.3
```

---

## shouldRasterize — when to use it

```swift
// USE: static, complex layer (e.g. a badge that doesn't change)
badgeLayer.shouldRasterize = true
badgeLayer.rasterizationScale = UIScreen.main.scale  // always set this!

// AVOID: animating layers, layers with changing content
// Rasterised cache is invalidated on every change — worse than nothing
```

---

## drawsAsynchronously — background draw

Safe for layers that draw into a CGContext (custom `CALayer.draw(in:)` subclasses):
```swift
contentLayer.drawsAsynchronously = true
```

Do NOT use with `CAShapeLayer`, `CAGradientLayer`, or any layer that manages its own
content — it applies only to layers that use `draw(in context:)`.

---

## Image loading and decoding

Never load a 12MP photo full-size just to display it at 100×100pt. Decode at display size:

```swift
func downsampledImage(at url: URL, to pointSize: CGSize, scale: CGFloat) -> UIImage? {
    let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    guard let source = CGImageSourceCreateWithURL(url as CFURL, imageSourceOptions) else { return nil }

    let maxDimensionInPixels = max(pointSize.width, pointSize.height) * scale
    let thumbnailOptions = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxDimensionInPixels
    ] as CFDictionary

    guard let thumbnail = CGImageSourceCreateThumbnailAtIndex(source, 0, thumbnailOptions) else { return nil }
    return UIImage(cgImage: thumbnail)
}

// Always decode off the main thread
Task.detached(priority: .userInitiated) {
    let image = downsampledImage(at: url, to: imageView.bounds.size, scale: UIScreen.main.scale)
    await MainActor.run { imageView.image = image }
}
```

---

## CAShapeLayer vs drawRect — when to use each

| Scenario | Use |
|---|---|
| Shape is animated (scale, position, strokeEnd) | `CAShapeLayer` |
| Shape is static but complex | `drawRect` or off-screen CGContext |
| Shape changes frequently (live drawing) | `CAShapeLayer` with `path` updates |
| Many identical shapes | `CAReplicatorLayer` + `CAShapeLayer` |
| Custom text + graphics together | `drawRect` (TextKit in the same pass) |

---

## Texture memory budget

| Device | Safe texture memory budget |
|---|---|
| iPhone 15 Pro | ~1.5 GB |
| iPhone 13 | ~800 MB |
| iPhone SE (3rd gen) | ~500 MB |
| iPad Pro M2 | ~2 GB |

A 4K (3840×2160) RGBA texture = 4 × 3840 × 2160 ≈ **31 MB**.
Keep the sum of all live textures under 50% of the device budget.
Use `MTLTexture.setPurgeableState(.volatile)` to allow the OS to reclaim textures
you don't immediately need.

---

## CADisplayLink — frame-accurate timer

```swift
class AnimationController {
    var displayLink: CADisplayLink?
    var startTime: CFTimeInterval = 0

    func start() {
        displayLink = CADisplayLink(target: self, selector: #selector(tick))
        displayLink?.preferredFrameRateRange = CAFrameRateRange(minimum: 60, maximum: 120)
        displayLink?.add(to: .main, forMode: .common)
        startTime = CACurrentMediaTime()
    }

    @objc func tick(link: CADisplayLink) {
        let elapsed = link.targetTimestamp - startTime
        // Update animation with elapsed
    }

    func stop() {
        displayLink?.invalidate()
        displayLink = nil
    }
}
```

---

## Memory management rules

- `CGContext`, `CGPath`, `CGGradient` — Swift ARC manages these; no manual release needed in Swift
- `MTLBuffer` / `MTLTexture` — ARC managed; set to `nil` to release
- `CVPixelBuffer` — use `CVPixelBufferLockBaseAddress` / `CVPixelBufferUnlockBaseAddress` in pairs
- `UIGraphicsBeginImageContext` — always paired with `UIGraphicsEndImageContext`; use `defer` to guarantee
- Prefer `UIGraphicsImageRenderer` over `UIGraphicsBeginImageContextWithOptions` — it handles memory and scale automatically
