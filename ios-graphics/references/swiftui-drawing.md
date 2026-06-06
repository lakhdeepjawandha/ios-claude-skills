# SwiftUI Drawing Reference

## Path — building custom shapes

```swift
// Custom path with arcs, lines, and curves
struct HeartShape: Shape {
    func path(in rect: CGRect) -> Path {
        Path { path in
            let w = rect.width, h = rect.height
            path.move(to: CGPoint(x: w / 2, y: h * 0.85))
            path.addCurve(
                to: CGPoint(x: 0, y: h * 0.25),
                control1: CGPoint(x: w * 0.1, y: h * 0.7),
                control2: CGPoint(x: 0, y: h * 0.5)
            )
            path.addArc(
                center: CGPoint(x: w * 0.25, y: h * 0.25),
                radius: w * 0.25,
                startAngle: .degrees(180),
                endAngle: .degrees(0),
                clockwise: false
            )
            path.addArc(
                center: CGPoint(x: w * 0.75, y: h * 0.25),
                radius: w * 0.25,
                startAngle: .degrees(180),
                endAngle: .degrees(0),
                clockwise: false
            )
            path.addCurve(
                to: CGPoint(x: w / 2, y: h * 0.85),
                control1: CGPoint(x: w, y: h * 0.5),
                control2: CGPoint(x: w * 0.9, y: h * 0.7)
            )
            path.closeSubpath()
        }
    }
}

// Usage
HeartShape()
    .fill(.red)
    .overlay(HeartShape().stroke(.white, lineWidth: 2))
    .frame(width: 120, height: 120)
```

## Canvas — imperative drawing surface

```swift
Canvas { context, size in
    // Resolved symbol (SF Symbol or image)
    if let symbol = context.resolveSymbol(id: "star") {
        context.draw(symbol, at: CGPoint(x: size.width / 2, y: size.height / 2))
    }

    // Gradient fill on a shape
    let gradient = Gradient(colors: [.blue, .purple])
    context.fill(
        Path(ellipseIn: CGRect(x: 20, y: 20, width: size.width - 40, height: size.height - 40)),
        with: .linearGradient(gradient, startPoint: .zero, endPoint: CGPoint(x: size.width, y: size.height))
    )

    // Clipping
    context.clip(to: Path(CGRect(x: 10, y: 10, width: size.width - 20, height: size.height - 20)))

    // Opacity layer
    context.withCGContext { cgCtx in
        // Drop to Core Graphics for anything Canvas doesn't support natively
        cgCtx.setFillColor(UIColor.red.cgColor)
        cgCtx.fill(CGRect(x: 0, y: 0, width: 50, height: 50))
    }
} symbols: {
    Image(systemName: "star.fill")
        .tag("star")
        .foregroundStyle(.yellow)
}
.frame(height: 200)
```

## Animatable custom shapes

```swift
struct AnimatedArc: Shape {
    var progress: Double  // 0.0 → 1.0

    var animatableData: Double {
        get { progress }
        set { progress = newValue }
    }

    func path(in rect: CGRect) -> Path {
        Path { path in
            path.addArc(
                center: CGPoint(x: rect.midX, y: rect.midY),
                radius: min(rect.width, rect.height) / 2 - 8,
                startAngle: .degrees(-90),
                endAngle: .degrees(-90 + 360 * progress),
                clockwise: false
            )
        }
    }
}

// Animated progress ring
struct ProgressRing: View {
    var progress: Double

    var body: some View {
        ZStack {
            Circle().stroke(.quaternary, lineWidth: 12)
            AnimatedArc(progress: progress)
                .stroke(.blue, style: StrokeStyle(lineWidth: 12, lineCap: .round))
                .animation(.spring(duration: 0.8), value: progress)
        }
    }
}
```

## drawingGroup — offscreen GPU compositing

Use when a view hierarchy has complex blending or many overlapping semi-transparent layers:

```swift
MyComplexView()
    .drawingGroup()  // flattens the hierarchy into a single Metal texture
```

**When to use**: 20+ layered views with opacity, particle-like effects, heavy blending.
**When NOT to use**: views with text (TextKit won't render into a Metal context), views
that need hit-testing on individual sub-layers.

## GeometryReader — size-dependent drawing

```swift
struct ResponsiveChart: View {
    var body: some View {
        GeometryReader { geo in
            let w = geo.size.width
            let h = geo.size.height
            Canvas { context, _ in
                // Draw chart relative to w, h
                let barWidth = w / 10
                for i in 0..<10 {
                    let barHeight = CGFloat.random(in: 20...h)
                    context.fill(
                        Path(CGRect(x: CGFloat(i) * barWidth, y: h - barHeight, width: barWidth - 4, height: barHeight)),
                        with: .color(.blue)
                    )
                }
            }
        }
        .frame(maxWidth: .infinity, maxHeight: 200)
    }
}
```

## TimelineView — time-driven animations

```swift
TimelineView(.animation) { timeline in
    let now = timeline.date.timeIntervalSinceReferenceDate
    Canvas { context, size in
        let angle = now.truncatingRemainder(dividingBy: 2) * .pi  // full rotation every 2s
        let x = size.width / 2 + 60 * cos(angle)
        let y = size.height / 2 + 60 * sin(angle)
        context.fill(Path(ellipseIn: CGRect(x: x - 10, y: y - 10, width: 20, height: 20)), with: .color(.orange))
    }
}
.frame(height: 160)
```

## Blend modes and effects

```swift
// Blend mode on a shape
Circle()
    .fill(.blue)
    .blendMode(.multiply)

// Visual effect (blur, saturation)
Rectangle()
    .fill(.clear)
    .background(.ultraThinMaterial)

// Canvas blend mode
context.blendMode = .screen
context.fill(myPath, with: .color(.white))
```
