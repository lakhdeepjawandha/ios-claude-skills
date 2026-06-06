# Core Graphics Reference

## CGContext — the fundamental drawing context

### Creating contexts
```swift
// Off-screen bitmap context (most common)
let renderer = UIGraphicsImageRenderer(size: CGSize(width: 200, height: 200))
let image = renderer.image { ctx in
    // ctx.cgContext is your CGContext
    ctx.cgContext.setFillColor(UIColor.red.cgColor)
    ctx.cgContext.fill(CGRect(x: 20, y: 20, width: 160, height: 160))
}

// PDF context
let pdfData = NSMutableData()
UIGraphicsBeginPDFContextToData(pdfData, CGRect(x: 0, y: 0, width: 595, height: 842), nil)
UIGraphicsBeginPDFPage()
// draw...
UIGraphicsEndPDFContext()
```

### Coordinate system flip (UIKit vs CG)
CG origin is bottom-left. When drawing into a UIKit view's `draw(_:)`, the context
is already flipped for you. In a raw bitmap context, flip manually:
```swift
context.translateBy(x: 0, y: size.height)
context.scaleBy(x: 1, y: -1)
```

### Path construction
```swift
let context = UIGraphicsGetCurrentContext()!

// Lines
context.move(to: CGPoint(x: 10, y: 10))
context.addLine(to: CGPoint(x: 200, y: 10))
context.addLine(to: CGPoint(x: 200, y: 200))
context.closePath()
context.setStrokeColor(UIColor.black.cgColor)
context.strokePath()

// Bezier curve
context.move(to: CGPoint(x: 10, y: 100))
context.addCurve(
    to: CGPoint(x: 300, y: 100),
    control1: CGPoint(x: 100, y: 10),
    control2: CGPoint(x: 200, y: 190)
)
context.strokePath()

// Arc
context.addArc(
    center: CGPoint(x: 150, y: 150),
    radius: 80,
    startAngle: 0,
    endAngle: .pi * 2,
    clockwise: false
)
context.fillPath()
```

### Fill and stroke styles
```swift
// Even-odd fill rule (for donut shapes)
context.setFillColor(UIColor.blue.cgColor)
context.fillPath(using: .evenOdd)

// Line styles
context.setLineWidth(3)
context.setLineCap(.round)       // .butt, .round, .square
context.setLineJoin(.miter)      // .miter, .round, .bevel
context.setLineDash(phase: 0, lengths: [10, 5])

// Shadow
context.setShadow(offset: CGSize(width: 2, height: 2), blur: 6, color: UIColor.black.withAlphaComponent(0.5).cgColor)
```

### Gradients
```swift
let colorSpace = CGColorSpaceCreateDeviceRGB()
let colors = [UIColor.red.cgColor, UIColor.blue.cgColor] as CFArray
let gradient = CGGradient(colorsSpace: colorSpace, colors: colors, locations: [0, 1])!

// Linear
context.drawLinearGradient(
    gradient,
    start: CGPoint(x: 0, y: 0),
    end: CGPoint(x: 200, y: 200),
    options: []
)

// Radial
context.drawRadialGradient(
    gradient,
    startCenter: CGPoint(x: 100, y: 100), startRadius: 0,
    endCenter: CGPoint(x: 100, y: 100), endRadius: 100,
    options: []
)
```

### Clipping
```swift
// Clip to a circle before drawing
context.addEllipse(in: CGRect(x: 0, y: 0, width: 200, height: 200))
context.clip()
// Everything after this is clipped to the ellipse
context.drawLinearGradient(...)
```

### Text rendering (for custom text in CG — prefer TextKit for complex text)
```swift
let attributes: [NSAttributedString.Key: Any] = [
    .font: UIFont.systemFont(ofSize: 18, weight: .bold),
    .foregroundColor: UIColor.white
]
let text = "Hello" as NSString
text.draw(at: CGPoint(x: 20, y: 20), withAttributes: attributes)
```

### CGImage manipulation
```swift
// Crop
let cropped = cgImage.cropping(to: CGRect(x: 10, y: 10, width: 100, height: 100))

// Scale
let renderer = UIGraphicsImageRenderer(size: targetSize)
let scaled = renderer.image { _ in
    UIImage(cgImage: cgImage).draw(in: CGRect(origin: .zero, size: targetSize))
}
```

## UIBezierPath — UIKit convenience wrapper

```swift
// Rounded rect
let path = UIBezierPath(roundedRect: CGRect(x: 0, y: 0, width: 200, height: 100), cornerRadius: 12)

// Star shape
func starPath(center: CGPoint, outerRadius: CGFloat, innerRadius: CGFloat, points: Int) -> UIBezierPath {
    let path = UIBezierPath()
    let step = CGFloat.pi / CGFloat(points)
    for i in 0 ..< points * 2 {
        let radius = i.isMultiple(of: 2) ? outerRadius : innerRadius
        let angle = CGFloat(i) * step - .pi / 2
        let point = CGPoint(x: center.x + radius * cos(angle), y: center.y + radius * sin(angle))
        i == 0 ? path.move(to: point) : path.addLine(to: point)
    }
    path.close()
    return path
}
```

## CGImageSource — efficient large image loading

```swift
let options: [String: Any] = [kCGImageSourceShouldCacheImmediately as String: false]
guard let source = CGImageSourceCreateWithURL(url as CFURL, nil),
      let cgImage = CGImageSourceCreateImageAtIndex(source, 0, options as CFDictionary) else { return nil }

// Thumbnail (decode at small size — never load a 20MP image full size just to display at 100pt)
let thumbOptions: [String: Any] = [
    kCGImageSourceThumbnailMaxPixelSize as String: 200,
    kCGImageSourceCreateThumbnailFromImageAlways as String: true
]
let thumb = CGImageSourceCreateThumbnailAtIndex(source, 0, thumbOptions as CFDictionary)
```
