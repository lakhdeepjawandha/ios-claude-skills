# Image Processing Reference (Core Image + vImage)

## Core Image — CIFilter pipeline

### Applying filters
```swift
import CoreImage
import CoreImage.CIFilterBuiltins

let context = CIContext()  // expensive to create — reuse across calls

// Type-safe filter API (iOS 13+)
let filter = CIFilter.gaussianBlur()
filter.inputImage = CIImage(image: inputUIImage)!
filter.radius = 20

if let output = filter.outputImage {
    // Crop to original extent (blur expands the image)
    let cropped = output.cropped(to: filter.inputImage!.extent)
    if let cgImage = context.createCGImage(cropped, from: cropped.extent) {
        let result = UIImage(cgImage: cgImage)
    }
}
```

### Chaining filters
```swift
let image = CIImage(image: sourceImage)!

let sharpen = CIFilter.unsharpMask()
sharpen.inputImage = image
sharpen.radius = 2.5
sharpen.intensity = 0.8

let vignette = CIFilter.vignette()
vignette.inputImage = sharpen.outputImage
vignette.intensity = 1.5
vignette.radius = 2.0

let finalImage = context.createCGImage(vignette.outputImage!, from: image.extent)
```

### Common filter catalogue
```swift
// Colour adjustments
CIFilter.colorControls()        // brightness, contrast, saturation
CIFilter.exposureAdjust()       // EV compensation
CIFilter.hueAdjust()            // hue rotation
CIFilter.highlightShadowAdjust() // tone mapping
CIFilter.vibrance()             // intelligent saturation

// Blur
CIFilter.gaussianBlur()
CIFilter.motionBlur()
CIFilter.zoomBlur()
CIFilter.bokehBlur()            // aperture-shaped bokeh (iOS 13+)
CIFilter.morphologyGradient()   // edge detection

// Distortion
CIFilter.bumpDistortion()
CIFilter.pinchDistortion()
CIFilter.twirlDistortion()

// Composite / blend
CIFilter.sourceOverCompositing()
CIFilter.multiplyCompositing()
CIFilter.screenBlendMode()

// Generator
CIFilter.qrCodeGenerator()      // barcode generation
CIFilter.checkerboardGenerator()
CIFilter.randomGenerator()      // noise
```

### Metal-backed CIContext (best performance)
```swift
let device = MTLCreateSystemDefaultDevice()!
let context = CIContext(mtlDevice: device)
```

### Render to Metal texture directly
```swift
context.render(ciImage,
               to: mtlTexture,
               commandBuffer: commandBuffer,
               bounds: ciImage.extent,
               colorSpace: CGColorSpaceCreateDeviceRGB())
```

---

## vImage — high-performance pixel manipulation

Use vImage when you need speed that CIFilter can't provide, or when you need
pixel-level access without GPU round-trips.

### Basic convolution (custom kernel)
```swift
import Accelerate

func applyConvolution(to sourceBuffer: vImage_Buffer, kernel: [Int16]) -> vImage_Buffer? {
    let kernelSide = Int(sqrt(Double(kernel.count)))
    var destBuffer = vImage_Buffer()
    vImageBuffer_Init(&destBuffer, sourceBuffer.height, sourceBuffer.width, 32, vImage_Flags(kvImageNoFlags))
    var mutableKernel = kernel
    vImageConvolve_ARGB8888(&sourceBuffer, &destBuffer, nil, 0, 0,
                             &mutableKernel, UInt32(kernelSide), UInt32(kernelSide),
                             0, nil, vImage_Flags(kvImageEdgeExtend))
    return destBuffer
}
```

### Fast resize (Lanczos)
```swift
func resize(cgImage: CGImage, to targetSize: CGSize) -> CGImage? {
    guard var sourceBuffer = try? vImage_Buffer(cgImage: cgImage) else { return nil }
    defer { sourceBuffer.free() }

    let destWidth = Int(targetSize.width)
    let destHeight = Int(targetSize.height)
    var destBuffer = vImage_Buffer()
    vImageBuffer_Init(&destBuffer, vImagePixelCount(destHeight), vImagePixelCount(destWidth), 32, vImage_Flags(kvImageNoFlags))
    defer { destBuffer.free() }

    vImageScale_ARGB8888(&sourceBuffer, &destBuffer, nil, vImage_Flags(kvImageHighQualityResampling))

    let format = vImage_CGImageFormat(bitsPerComponent: 8, bitsPerPixel: 32,
                                       colorSpace: CGColorSpaceCreateDeviceRGB(),
                                       bitmapInfo: CGBitmapInfo(rawValue: CGImageAlphaInfo.first.rawValue))!
    return try? destBuffer.createCGImage(format: format)
}
```

### Histogram equalization
```swift
var mutableBuffer = sourceBuffer
vImageEqualization_ARGB8888(&mutableBuffer, &destBuffer, vImage_Flags(kvImageNoFlags))
```
