# Core Animation Reference

## CALayer fundamentals

### Layer hierarchy and properties
```swift
let layer = CALayer()
layer.frame = CGRect(x: 0, y: 0, width: 100, height: 100)
layer.backgroundColor = UIColor.systemBlue.cgColor
layer.cornerRadius = 12
layer.borderWidth = 2
layer.borderColor = UIColor.white.cgColor
layer.shadowOffset = CGSize(width: 0, height: 4)
layer.shadowRadius = 8
layer.shadowColor = UIColor.black.cgColor
layer.shadowOpacity = 0.3
view.layer.addSublayer(layer)
```

### anchorPoint — the rotation/scale pivot
Default is (0.5, 0.5) — the centre. Range [0,1].
```swift
layer.anchorPoint = CGPoint(x: 0, y: 0)   // top-left pivot
layer.anchorPoint = CGPoint(x: 0.5, y: 1) // bottom-centre pivot (for door-hinge effects)
// WARNING: changing anchorPoint moves the layer visually — compensate with position:
// layer.position stays the same screen point, but the anchor shifts
```

## Animation types

### CABasicAnimation
```swift
let anim = CABasicAnimation(keyPath: "opacity")
anim.fromValue = 1.0
anim.toValue = 0.0
anim.duration = 0.5
anim.autoreverses = true
anim.repeatCount = .infinity
layer.add(anim, forKey: "fade")
```

### CAKeyframeAnimation
```swift
let bounce = CAKeyframeAnimation(keyPath: "transform.translation.y")
bounce.values = [0, -30, -15, -25, -20, -22, -20]
bounce.keyTimes = [0, 0.2, 0.35, 0.5, 0.65, 0.8, 1.0]
bounce.duration = 0.8
bounce.timingFunctions = [
    CAMediaTimingFunction(name: .easeIn),
    CAMediaTimingFunction(name: .easeOut),
    CAMediaTimingFunction(name: .easeIn),
    CAMediaTimingFunction(name: .easeOut),
    CAMediaTimingFunction(name: .easeIn),
    CAMediaTimingFunction(name: .easeOut)
]
layer.add(bounce, forKey: "bounce")
```

### CASpringAnimation
```swift
let spring = CASpringAnimation(keyPath: "transform.scale")
spring.fromValue = 0.8
spring.toValue = 1.0
spring.mass = 1
spring.stiffness = 200
spring.damping = 10
spring.duration = spring.settlingDuration
layer.add(spring, forKey: "spring")
```

### CAAnimationGroup
```swift
let fadeIn = CABasicAnimation(keyPath: "opacity")
fadeIn.fromValue = 0; fadeIn.toValue = 1

let moveUp = CABasicAnimation(keyPath: "position.y")
moveUp.fromValue = layer.position.y + 20
moveUp.toValue = layer.position.y

let group = CAAnimationGroup()
group.animations = [fadeIn, moveUp]
group.duration = 0.4
group.fillMode = .forwards
group.isRemovedOnCompletion = false
layer.add(group, forKey: "entrance")
```

## CATransaction — controlling implicit animations

```swift
CATransaction.begin()
CATransaction.setAnimationDuration(0.8)
CATransaction.setAnimationTimingFunction(CAMediaTimingFunction(name: .easeInEaseOut))
CATransaction.setCompletionBlock {
    print("Animation finished")
}
layer.position = CGPoint(x: 200, y: 300)
CATransaction.commit()

// Disable implicit animations
CATransaction.begin()
CATransaction.setDisableActions(true)
layer.frame = newFrame
CATransaction.commit()
```

## Specialised layer subclasses

### CAShapeLayer
```swift
let shapeLayer = CAShapeLayer()
shapeLayer.path = UIBezierPath(ovalIn: CGRect(x: 0, y: 0, width: 100, height: 100)).cgPath
shapeLayer.fillColor = UIColor.systemPurple.cgColor
shapeLayer.strokeColor = UIColor.white.cgColor
shapeLayer.lineWidth = 3

// Animated stroke draw-on effect
let drawAnim = CABasicAnimation(keyPath: "strokeEnd")
drawAnim.fromValue = 0
drawAnim.toValue = 1
drawAnim.duration = 1.2
shapeLayer.add(drawAnim, forKey: "draw")
```

### CAGradientLayer
```swift
let gradientLayer = CAGradientLayer()
gradientLayer.frame = view.bounds
gradientLayer.colors = [UIColor.systemBlue.cgColor, UIColor.systemPurple.cgColor]
gradientLayer.locations = [0, 1]
gradientLayer.startPoint = CGPoint(x: 0, y: 0)
gradientLayer.endPoint = CGPoint(x: 1, y: 1)
gradientLayer.type = .axial // .radial, .conic
view.layer.addSublayer(gradientLayer)
```

### CATextLayer
```swift
let textLayer = CATextLayer()
textLayer.string = "Hello, Metal"
textLayer.font = CTFontCreateWithName("Helvetica-Bold" as CFString, 24, nil)
textLayer.fontSize = 24
textLayer.foregroundColor = UIColor.white.cgColor
textLayer.contentsScale = UIScreen.main.scale  // IMPORTANT: avoid blurry text
textLayer.frame = CGRect(x: 10, y: 10, width: 200, height: 40)
view.layer.addSublayer(textLayer)
```

### CAEmitterLayer (particle effects)
```swift
let emitter = CAEmitterLayer()
emitter.emitterPosition = CGPoint(x: view.bounds.midX, y: 0)
emitter.emitterShape = .line
emitter.emitterSize = CGSize(width: view.bounds.width, height: 1)

let cell = CAEmitterCell()
cell.birthRate = 10
cell.lifetime = 6
cell.velocity = 100
cell.velocityRange = 50
cell.emissionLongitude = .pi  // downward
cell.scale = 0.1
cell.scaleRange = 0.05
cell.contents = UIImage(named: "snowflake")?.cgImage
emitter.emitterCells = [cell]
view.layer.addSublayer(emitter)
```

## The presentation layer

The **model layer** holds the final value. The **presentation layer** holds the
in-flight animated value. Hit-test against the presentation layer during animation:

```swift
let presentationLayer = layer.presentation()
let point = presentationLayer?.convert(touchPoint, from: nil)
```

## Transforms (CATransform3D)

```swift
// 2D transforms (also work as 3D with z=0)
layer.transform = CATransform3DMakeRotation(.pi / 4, 0, 0, 1)  // 45° rotation
layer.transform = CATransform3DMakeScale(1.5, 1.5, 1)
layer.transform = CATransform3DMakeTranslation(50, 0, 0)

// 3D perspective flip
var transform = CATransform3DIdentity
transform.m34 = -1 / 500  // perspective depth
transform = CATransform3DRotate(transform, .pi / 3, 0, 1, 0)  // Y-axis rotation
layer.transform = transform
```
