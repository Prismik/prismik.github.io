+++
date = '2025-07-05'
draft = false
title = 'SwiftTracer - A physically based rendering engine'
[params]
  math = true
+++

SwiftTracer is an implementation of a physically based rendering engine that is inspired by PBRT, Mitsuba and many other contributors in the field. At it's core, it uses [simd](https://developer.apple.com/documentation/accelerate/simd-library) to perform the ray tracing computations: mostly vectors and matrices operations along with some trigonometry. The tracing is done in parallel, thanks to the new `async/await` [concurrency model](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/) introduced in Swift 5.5. We split our target image in a series of blocks, all of which run a self-contained tracing task. The collision detection is done using a combination of shape intersections, AABBs and even acceleration techniques like BVH. Several image formats like png, jpg, pfm and exr are supported.


## Materials

There are currently 4 base materials implemented, which use their own distribution functions to support light scattering. On top of these, it is possible to blend them with a linear combination of two different materials. The strenght of the blend can be defined by an `alpha` parameter.

| Diffuse | Smooth metal | Rough metal | Glass | Blend | 
| :-: | :-: | :-: | :-: | :-: | 
|  ![diffuse](images/diffuse.png) | ![metal](images/metal.png)  | ![rough metal](images/metal-rough.png)  | ![glass](images/glass.png) | ![glass](images/blend.png) |

There are a few geometry primitives that are supported, although meshes and triangles is what have been used the most throughout my test scenes. We also have `quads` and `spheres`

Triangle intersection uses the [Möller–Trumbore ray-triangle intersection algorithm](https://dl.acm.org/doi/10.1145/1198555.1198746). Vertices and normals are currently supported. Tangents will be added at a later time when I port over anisotropic materials from my Rust implementation.

## Ray tracing

The rendering equation describes the propagation luminance quantity $L$ as it travels from any number of incoming and outgoing rays of light $\omega_i, \omega_o$.

$L_o(x, \omega_o) = L_e(x, \omega_o)+\int_{\Omega} f_v(x, \omega_i, \omega_o) L_i(x, \omega_i) (\omega_i \cdot n)\mathrm{d}\omega_i,$

where it's recursive application from surface to surface gives $F(\overline{x})$, the contribution for path $\overline{x}$. With $n$ interactions, we would have a list of $n+1$ vertices $\overline{x} = \{ x_0, ..., x_n \}$. This equation is typically solved by replacing the ingetrals with numerical methods.

Interactions are calculated in the local coordinate system, where both the incoming and outgoing $\omega_o, \omega_i$ directions are pointing outwards from a surface with a normal of $(0, 0, 1)$.

### Integrators

### Asynchronous processing

In order to render my scenes, I create a global task which awaits a series of block renders defined in `renderBlocks`.

```swift
func render(scene: Scene) -> PixelBuffer {
    let image = PixelBuffer(
        width: Int(scene.camera.resolution.x), 
        height: Int(scene.camera.resolution.y), 
        value: .zero
    )
    let gcd = DispatchGroup()
    gcd.enter()
    Task {
        defer { gcd.leave() }
        let blocks = await renderBlocks(blockSize: 32)
        return blocks.assemble(into: image)
    }
    gcd.wait()
    return image
}
```

The creation of the individual tasks happen in the `renderBlocks` function where they are added to the group, once per block. Depending on the resolution `res` of the resulting image and the chosen `blockSize`, there will be a varying number of spawned child processes. Their results are aggregated together using an `await` keyword, and once all rendering tasks are completed, their combined results is returned.

```swift
func renderBlocks(blockSize: Int, res: Vec2) async -> [Block] {
    return await withTaskGroup(of: Block.self) { group in
        for x in stride(from: 0, to: res.x, by: blockSize) {
            for y in stride(from: 0, to: res.y, by: blockSize) {
                let size = Vec2(
                    min(res.x - Float(x), Float(blockSize)),
                    min(res.y - Float(y), Float(blockSize))
                )
                
                group.addTask {
                    return integrate(size: size, pos: Vec2(x, y))
                }
            }
        }
        
        var blocks: [Block] = []
        for await block in group {
            blocks.append(block)
        }
        
        return blocks
    }
}
```

A simple rendering method that I have implemented is the `Path` integrator which uses a simple Monte-Carlo algorithm. The way it goes is that for each pixels $(x, y)$ in the block, we will cast $n$ rays of light and average their total contributions by the number of samples $n$.

```swift
func integrate(size: Vec2, pos: Vec2, samples n: Int) -> Block {
    let img = PixelBuffer(
        width: size.x, 
        height: size.y, 
        value: .zero
    )
    var block = Block(position: Vec2(x, y), size: size, image: img)
    
    for lx in 0 ..< block.size.x {
        for ly in 0 ..< block.size.y {
            let x = lx + block.position.x
            let y = ly + block.position.y
            
            var avg = Color()
            for _ in (0 ..< n) {
                // Anti aliasing
                let aaPos = Vec2(x, y) + sampler.next2()
                // Integrate the ray contribution
                avg += li(pixel: aaPos)
            }
            
            img[lx, ly] = avg / Float(n)
        }
    }
    
    block.image = img
    return block
}
```