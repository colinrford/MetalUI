# MetalUI
Metal with SwiftUI

MetalUI provides a basic utility for integrating Metal into SwiftUI. Since this was originally committed by the original author, Apple has added limited support for accessing Metal shaders. The interested reader may find more info in Apple Developer Documentation, [SwiftUI / Drawing and Graphics / Shader](https://developer.apple.com/documentation/swiftui/shader), in section Accessing Metal Shaders. From the following excerpt:

>Shaders also conform to the ShapeStyle protocol, letting their MSL shader function provide per-pixel color to fill any shape or text view. For a shader function to act as a fill pattern it must have a function signature matching:
>
```
[[ stitchable ]] half4 name(float2 position, args...)
```
>
>where position is the user-space coordinates of the pixel applied to the shader [...]

and a cursory scan (by the forker), it would seem shader support is limited to 2D. One could conceivably do gymnastics to circumvent this limitation, or one could try MetalUI to implement 3D graphics using Metal into a SwiftUI context. With MetalUI interfacing with arbitrary shaders – like `vertex`, `fragment`, or `mesh` – is made fairly straightforward. Example usage (provided by original author) may be found below.

After the example usage, the reader may find more commentary from the forker.

---------------------------------------------------------------

## Example Usage

### SwiftUI
```swift
import MetalUI
import SwiftUI

struct ContentView: View {
    var body: some View {
        MetalView {
            BasicMetalView()
        }
    }
}

```

### BasicMetalView
```swift
import MetalUI
import MetalKit

class BasicMetalView: MTKView, MetalPresenting {
    var renderer: MetalRendering!

    required init() {
        super.init(frame: .zero, device: MTLCreateSystemDefaultDevice())
        configure(device: device)
    }

    required init(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    func configureMTKView() {
        colorPixelFormat = .bgra8Unorm
        // Our clear color, can be set to any color
        clearColor = MTLClearColor(red: 1, green: 0.57, blue: 0.25, alpha: 1)
    }

    func renderer(forDevice device: MTLDevice) -> MetalRendering {
        BasicMetalRenderer(vertices: [
            MetalRenderingVertex(position: SIMD3(0,1,0), color: SIMD4(1,0,0,1)),
            MetalRenderingVertex(position: SIMD3(-1,-1,0), color: SIMD4(0,1,0,1)),
            MetalRenderingVertex(position: SIMD3(1,-1,0), color: SIMD4(0,0,1,1))
        ], device: device)
    }
}
```

### BasicMetalRenderer
```swift
import MetalUI
import MetalKit

final class BasicMetalRenderer: NSObject, MetalRendering {
    var commandQueue: MTLCommandQueue?
    var renderPipelineState: MTLRenderPipelineState?
    var vertexBuffer: MTLBuffer?

    var vertices: [MetalRenderingVertex] = []

    func createCommandQueue(device: MTLDevice) {
        commandQueue = device.makeCommandQueue()
    }

    func createPipelineState(
        withLibrary library: MTLLibrary?,
        forDevice device: MTLDevice
    ) {
        // Our vertex function name
        let vertexFunction = library?.makeFunction(name: "basic_vertex_function")
        // Our fragment function name
        let fragmentFunction = library?.makeFunction(name: "basic_fragment_function")
        // Create basic descriptor
        let renderPipelineDescriptor = MTLRenderPipelineDescriptor()
        // Attach the pixel format that si the same as the MetalView
        renderPipelineDescriptor.colorAttachments[0].pixelFormat = .bgra8Unorm
        // Attach the shader functions
        renderPipelineDescriptor.vertexFunction = vertexFunction
        renderPipelineDescriptor.fragmentFunction = fragmentFunction
        // Try to update the state of the renderPipeline
        do {
            renderPipelineState = try device.makeRenderPipelineState(descriptor: renderPipelineDescriptor)
        } catch {
            print(error.localizedDescription)
        }
    }

    func createBuffers(device: MTLDevice) {
        vertexBuffer = device.makeBuffer(bytes: vertices,
                                         length: MemoryLayout<MetalRenderingVertex>.stride * vertices.count,
                                         options: [])
    }

    func mtkView(_ view: MTKView, drawableSizeWillChange size: CGSize) {

    }

    func draw(in view: MTKView) {
        // Get the current drawable and descriptor
        guard let drawable = view.currentDrawable,
              let renderPassDescriptor = view.currentRenderPassDescriptor,
              let commandQueue = commandQueue,
              let renderPipelineState = renderPipelineState else {
            return
        }
        // Create a buffer from the commandQueue
        let commandBuffer = commandQueue.makeCommandBuffer()
        let commandEncoder = commandBuffer?.makeRenderCommandEncoder(descriptor: renderPassDescriptor)
        commandEncoder?.setRenderPipelineState(renderPipelineState)
        // Pass in the vertexBuffer into index 0
        commandEncoder?.setVertexBuffer(vertexBuffer, offset: 0, index: 0)
        // Draw primitive at vertextStart 0
        commandEncoder?.drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: vertices.count)

        commandEncoder?.endEncoding()
        commandBuffer?.present(drawable)
        commandBuffer?.commit()
    }
}
```

### Shaders.metal
```metal
#include <metal_stdlib>
using namespace metal;

struct VertexIn {
    float3 position;
    float4 color;
};
struct VertexOut {
    float4 position [[ position ]];
    float4 color;
};
vertex VertexOut basic_vertex_function(const device VertexIn *vertices [[ buffer(0) ]],
                                       uint vertexID [[ vertex_id  ]]) {
    VertexOut vOut;
    vOut.position = float4(vertices[vertexID].position,1);
    vOut.color = vertices[vertexID].color;
    return vOut;
}
fragment float4 basic_fragment_function(VertexOut vIn [[ stage_in ]]) {
    return vIn.color;
}
```

-------------------------------------------------------------

### Possible Modifications

Often when using Metal, one uses MetalKit as well. MetalKit provides two utilities to setup a view: the `MTKView` class and `MTKViewDelegate` protocol. In MetalUI one creates an `MTKView` by creating a class that conforms to the `MetalPresenting` protocol, and an `MTKViewDelegate` by creating a class that conforms to `MetalRendering` (which conforms to `MTKViewDelegate`). The `MetalPresenting` protocol seems more or less sufficient for most needs. The `MetalRendering` protocol on the other hand might benefit from some modifications.

If one wants to use more complicated renderers and rendering techniques, it might make sense to change the `MetalRendering` protocol (or add an additional protocol). As provided, it requires a `MTLRenderPipelineState`, `MTLBuffer`, and an array of the aptly-named `MetalRenderingVertex`. Carrying more pipeline states and buffers is somewhat of a syntactic issue, and hence rather easy to address, but nevertheless one will easily encounter inconveniences of the basic setup provided by MetalUI when working with more complicated renderers.

The forker does not feel strongly enough at this time to make any firm decision to alter the package here. Rather, just wanted to throw these thoughts out there. 

### Exercise for the reader

Make a presenter and renderer for an [`MTKMesh`](https://developer.apple.com/documentation/metalkit/mtkmesh)[^1].

[^1]: This is possible. It is not too hard and is an easy way to showcase the utility of this package. For the lazy reader, seek help from a generator...
