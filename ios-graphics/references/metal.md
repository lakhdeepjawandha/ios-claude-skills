# Metal Reference

## Setup — device, queue, layer

```swift
import Metal
import MetalKit

class MetalRenderer: NSObject, MTKViewDelegate {
    let device: MTLDevice
    let commandQueue: MTLCommandQueue
    var pipelineState: MTLRenderPipelineState!

    init(mtkView: MTKView) {
        guard let device = MTLCreateSystemDefaultDevice() else {
            fatalError("Metal not supported")
        }
        self.device = device
        self.commandQueue = device.makeCommandQueue()!
        mtkView.device = device
        mtkView.clearColor = MTLClearColorMake(0.1, 0.1, 0.15, 1.0)
        super.init()
        buildPipeline(view: mtkView)
    }

    func buildPipeline(view: MTKView) {
        let library = device.makeDefaultLibrary()!
        let vertexFn = library.makeFunction(name: "vertex_main")
        let fragmentFn = library.makeFunction(name: "fragment_main")

        let descriptor = MTLRenderPipelineDescriptor()
        descriptor.vertexFunction = vertexFn
        descriptor.fragmentFunction = fragmentFn
        descriptor.colorAttachments[0].pixelFormat = view.colorPixelFormat

        pipelineState = try! device.makeRenderPipelineState(descriptor: descriptor)
    }

    func draw(in view: MTKView) {
        guard let drawable = view.currentDrawable,
              let descriptor = view.currentRenderPassDescriptor else { return }

        let commandBuffer = commandQueue.makeCommandBuffer()!
        let encoder = commandBuffer.makeRenderCommandEncoder(descriptor: descriptor)!
        encoder.setRenderPipelineState(pipelineState)
        encoder.drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: 3)
        encoder.endEncoding()

        commandBuffer.present(drawable)
        commandBuffer.commit()
    }

    func mtkView(_ view: MTKView, drawableSizeWillChange size: CGSize) {}
}
```

## Metal Shading Language (MSL) — basic shader

```metal
// Shaders.metal
#include <metal_stdlib>
using namespace metal;

struct VertexOut {
    float4 position [[position]];
    float4 color;
};

// Hardcoded triangle — no vertex buffer needed for demos
vertex VertexOut vertex_main(uint vertexID [[vertex_id]]) {
    float2 positions[3] = {
        float2(0,  0.7),
        float2(-0.7, -0.7),
        float2(0.7, -0.7)
    };
    float3 colors[3] = {
        float3(1, 0, 0),
        float3(0, 1, 0),
        float3(0, 0, 1)
    };
    VertexOut out;
    out.position = float4(positions[vertexID], 0, 1);
    out.color = float4(colors[vertexID], 1);
    return out;
}

fragment float4 fragment_main(VertexOut in [[stage_in]]) {
    return in.color;
}
```

## Vertex buffers

```swift
struct Vertex {
    var position: SIMD2<Float>
    var color: SIMD4<Float>
}

let vertices: [Vertex] = [
    Vertex(position: [0, 0.5],   color: [1, 0, 0, 1]),
    Vertex(position: [-0.5, -0.5], color: [0, 1, 0, 1]),
    Vertex(position: [0.5, -0.5],  color: [0, 0, 1, 1])
]

let vertexBuffer = device.makeBuffer(
    bytes: vertices,
    length: MemoryLayout<Vertex>.stride * vertices.count,
    options: .storageModeShared
)

// In draw():
encoder.setVertexBuffer(vertexBuffer, offset: 0, index: 0)
encoder.drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: 3)
```

## Uniforms (passing data to shaders)

```swift
struct Uniforms {
    var time: Float
    var resolution: SIMD2<Float>
}

var uniforms = Uniforms(time: Float(Date().timeIntervalSince1970), resolution: [800, 600])
encoder.setFragmentBytes(&uniforms, length: MemoryLayout<Uniforms>.size, index: 0)
```

```metal
struct Uniforms {
    float time;
    float2 resolution;
};

fragment float4 fragment_main(VertexOut in [[stage_in]],
                               constant Uniforms &u [[buffer(0)]]) {
    float2 uv = in.position.xy / u.resolution;
    float3 col = 0.5 + 0.5 * cos(u.time + uv.xyx + float3(0, 2, 4));
    return float4(col, 1.0);
}
```

## Textures

```swift
// Load texture from UIImage
func makeTexture(from image: UIImage, device: MTLDevice) -> MTLTexture? {
    let loader = MTKTextureLoader(device: device)
    guard let cgImage = image.cgImage else { return nil }
    return try? loader.newTexture(cgImage: cgImage, options: [
        .textureUsage: MTLTextureUsage.shaderRead.rawValue as NSObject,
        .generateMipmaps: true as NSObject
    ])
}

// In draw():
encoder.setFragmentTexture(texture, index: 0)
encoder.setFragmentSamplerState(samplerState, index: 0)
```

```metal
fragment float4 fragment_main(VertexOut in [[stage_in]],
                               texture2d<float> tex [[texture(0)]],
                               sampler samp [[sampler(0)]]) {
    return tex.sample(samp, in.texcoord);
}
```

## Compute kernels (GPU parallel compute)

```swift
let library = device.makeDefaultLibrary()!
let computeFn = library.makeFunction(name: "process_image")!
let computePipeline = try! device.makeComputePipelineState(function: computeFn)

let commandBuffer = commandQueue.makeCommandBuffer()!
let encoder = commandBuffer.makeComputeCommandEncoder()!
encoder.setComputePipelineState(computePipeline)
encoder.setTexture(inputTexture, index: 0)
encoder.setTexture(outputTexture, index: 1)

let threadsPerGroup = MTLSize(width: 8, height: 8, depth: 1)
let threadGroups = MTLSize(
    width: (inputTexture.width + 7) / 8,
    height: (inputTexture.height + 7) / 8,
    depth: 1
)
encoder.dispatchThreadgroups(threadGroups, threadsPerThreadgroup: threadsPerGroup)
encoder.endEncoding()
commandBuffer.commit()
```

```metal
kernel void process_image(texture2d<float, access::read> input [[texture(0)]],
                           texture2d<float, access::write> output [[texture(1)]],
                           uint2 gid [[thread_position_in_grid]]) {
    float4 color = input.read(gid);
    // Invert colours
    float4 result = float4(1.0 - color.rgb, color.a);
    output.write(result, gid);
}
```

## SwiftUI + Metal via MTKView

```swift
struct MetalView: UIViewRepresentable {
    func makeUIView(context: Context) -> MTKView {
        let view = MTKView()
        view.delegate = context.coordinator
        view.device = MTLCreateSystemDefaultDevice()
        view.preferredFramesPerSecond = 60
        return view
    }

    func updateUIView(_ uiView: MTKView, context: Context) {}

    func makeCoordinator() -> MetalRenderer {
        MetalRenderer(mtkView: makeUIView(context: .init(coordinator: MetalRenderer(mtkView: MTKView()))))
    }
}
```

## Performance tips
- **Triple buffering**: use a semaphore + 3 uniform buffers to avoid GPU/CPU stalls
- **Blit encoder**: use `MTLBlitCommandEncoder` for texture copies; never read back GPU→CPU in the render loop
- **Indirect command buffers**: encode draw calls on the GPU itself for very high object counts
- **Argument buffers**: group many resources into a single struct for efficient binding
- **GPU Frame Capture**: Xcode → Debug → Capture GPU Frame — invaluable for shader debugging
