# Create a triangle with rendy

This is a quick walkthrough of the rendy [triangle example](https://github.com/amethyst/rendy/blob/release-0.5.1/rendy/examples/triangle/main.rs).

\#S:INCLUDE

These are all the imports we need.
```rust
use rendy::{
    command::{Families, QueueId, RenderPassEncoder},
    factory::{Config, Factory},
    graph::{render::*, Graph, GraphBuilder, GraphContext, NodeBuffer, NodeImage},
    hal::{self, Backend},
    init::winit::{
        event::{Event, WindowEvent},
        event_loop::{ControlFlow, EventLoop},
        window::WindowBuilder,
    },
    init::AnyWindowedRendy,
    memory::Dynamic,
    mesh::PosColor,
    resource::{Buffer, BufferInfo, DescriptorSetLayout, Escape, Handle},
    shader::{ShaderKind, SourceLanguage, SourceShaderInfo, SpirvShader},
};
```
```rust
#[cfg(feature = "spirv-reflection")]
use rendy::shader::SpirvReflection;

#[cfg(not(feature = "spirv-reflection"))]
use rendy::mesh::AsVertex;
```
Pulls in vertex shader source and compiles it.
```rust
lazy_static::lazy_static! {
    static ref VERTEX: SpirvShader = SourceShaderInfo::new(
        include_str!("shader.vert"),
        concat!(env!("CARGO_MANIFEST_DIR"), "/src/shader.vert").into(),
        ShaderKind::Vertex,
        SourceLanguage::GLSL,
        "main",
    ).precompile().unwrap();

```
Pulls in fragment shader source and compiles it.
```rust
    static ref FRAGMENT: SpirvShader = SourceShaderInfo::new(
        include_str!("shader.frag"),
        concat!(env!("CARGO_MANIFEST_DIR"), "/src/shader.frag").into(),
        ShaderKind::Fragment,
        SourceLanguage::GLSL,
        "main",
    ).precompile().unwrap();

    static ref SHADERS: rendy::shader::ShaderSetBuilder = rendy::shader::ShaderSetBuilder::default()
        .with_vertex(&*VERTEX).unwrap()
        .with_fragment(&*FRAGMENT).unwrap();
}

```
We need to reflect the shaders so we can use the types when we build the pipeline.
```rust
#[cfg(feature = "spirv-reflection")]
lazy_static::lazy_static! {
    static ref SHADER_REFLECTION: SpirvReflection = SHADERS.reflect().unwrap();
}

```
These types are used to describe our render pipeline. They will be set up later using the impl blocks.
```rust
#[derive(Debug, Default)]
struct TriangleRenderPipelineDesc;

#[derive(Debug)]
struct TriangleRenderPipeline<B: hal::Backend> {
    vertex: Option<Escape<Buffer<B>>>,
}

```
The pipeline description is used to tell the pipeline about the shaders.
```rust
impl<B, T> SimpleGraphicsPipelineDesc<B, T> for TriangleRenderPipelineDesc
where
    B: hal::Backend,
    T: ?Sized,
{
    type Pipeline = TriangleRenderPipeline<B>;

    fn depth_stencil(&self) -> Option<hal::pso::DepthStencilDesc> {
        None
    }

```
Load both the shaders in.
```rust
    fn load_shader_set(&self, factory: &mut Factory<B>, _aux: &T) -> rendy::shader::ShaderSet<B> {
        SHADERS.build(factory, Default::default()).unwrap()
    }

```
The vertices are described using types in the shader.
```rust
    fn vertices(
        &self,
    ) -> Vec<(
        Vec<hal::pso::Element<hal::format::Format>>,
        hal::pso::ElemStride,
        hal::pso::VertexInputRate,
    )> {
        #[cfg(feature = "spirv-reflection")]
        return vec![SHADER_REFLECTION
            .attributes_range(..)
            .unwrap()
            .gfx_vertex_input_desc(hal::pso::VertexInputRate::Vertex)];

        #[cfg(not(feature = "spirv-reflection"))]
        return vec![PosColor::vertex().gfx_vertex_input_desc(hal::pso::VertexInputRate::Vertex)];
    }

    fn build<'a>(
        self,
        _ctx: &GraphContext<B>,
        _factory: &mut Factory<B>,
        _queue: QueueId,
        _aux: &T,
        buffers: Vec<NodeBuffer>,
        images: Vec<NodeImage>,
        set_layouts: &[Handle<DescriptorSetLayout<B>>],
    ) -> Result<TriangleRenderPipeline<B>, rendy::core::hal::pso::CreationError> {
        assert!(buffers.is_empty());
        assert!(images.is_empty());
        assert!(set_layouts.is_empty());

        Ok(TriangleRenderPipeline { vertex: None })
    }
}

```
Create a graphics pipeline.
```rust
impl<B, T> SimpleGraphicsPipeline<B, T> for TriangleRenderPipeline<B>
where
    B: hal::Backend,
    T: ?Sized,
{
    type Desc = TriangleRenderPipelineDesc;

```
### Prepare this pipeline
Prepare the graphics pipeline. Note that only the factory is used.  
There's lots that could be configured but this is example is desgined to keep things simple.
```rust
    fn prepare(
        &mut self,
        factory: &Factory<B>,
        _queue: QueueId,
        _set_layouts: &[Handle<DescriptorSetLayout<B>>],
        _index: usize,
        _aux: &T,
    ) -> PrepareResult {
        if self.vertex.is_none() {
```
Extract the vertex buffer size from the shader reflection.
```rust
            #[cfg(feature = "spirv-reflection")]
            let vbuf_size = SHADER_REFLECTION.attributes_range(..).unwrap().stride as u64 * 3;

            #[cfg(not(feature = "spirv-reflection"))]
            let vbuf_size = PosColor::vertex().stride as u64 * 3;

```
Create a buffer for the verticies to use.
```rust
            let mut vbuf = factory
                .create_buffer(
                    BufferInfo {
                        size: vbuf_size,
                        usage: hal::buffer::Usage::VERTEX,
                    },
                    Dynamic,
                )
                .unwrap();

```
Write data into the vertex buffer.
```rust
            unsafe {
                // Fresh buffer.
                factory
                    .upload_visible_buffer(
                        &mut vbuf,
                        0,
                        &[
                            PosColor {
```
The colors are interpolated from each vertex.
Create a red vertex at x: 0.0, y: -0.5, z: 0.0.
```rust
                                position: [0.0, -0.5, 0.0].into(),
                                color: [1.0, 0.0, 0.0, 1.0].into(),
                            },
```
Create a green vertex at x: 0.5, y: 0.5, z: 0.0.
```rust
                            PosColor {
                                position: [0.5, 0.5, 0.0].into(),
                                color: [0.0, 1.0, 0.0, 1.0].into(),
                            },
```
Create a blue vertex at x: -0.5, y: 0.5, z: 0.0.
```rust
                            PosColor {
                                position: [-0.5, 0.5, 0.0].into(),
                                color: [0.0, 0.0, 1.0, 1.0].into(),
                            },
                        ],
                    )
                    .unwrap();
            }

            self.vertex = Some(vbuf);
        }

```
Tells rendy that this will be reused so it can be cached in memory.
```rust
        PrepareResult::DrawReuse
    }

```
### Draw this pipeline
The draw function will be called each frame.
```rust
    fn draw(
        &mut self,
        _layout: &B::PipelineLayout,
        mut encoder: RenderPassEncoder<'_, B>,
        _index: usize,
        _aux: &T,
    ) {
```
Get the vertex buffer and bind it too the render pass.
```rust
        let vbuf = self.vertex.as_ref().unwrap();
        unsafe {
            encoder.bind_vertex_buffers(0, Some((vbuf.raw(), 0)));
```
Draw the verticies from 0 to 2. (0..3 is a range 0, 1, 2).
Use 1 instance.
```rust
            encoder.draw(0..3, 0..1);
        }
    }

    fn dispose(self, _factory: &mut Factory<B>, _aux: &T) {}
}


## Run The loop
```
This function will be used to run the whole program loop.
```rust
fn run<B: Backend>(
    event_loop: EventLoop<()>,
    mut factory: Factory<B>,
    mut families: Families<B>,
    graph: Graph<B, ()>,
) {
    let started = std::time::Instant::now();

```
Gather frame data.
```rust
    let mut frame = 0u64;
    let mut elapsed = started.elapsed();
```
Get the graph
```rust
    let mut graph = Some(graph);

```
Use the winit loop to run the program.
```rust
    event_loop.run(move |event, _, control_flow| {
        *control_flow = ControlFlow::Poll;
        match event {
```
Check for a window close event and exit.
```rust
            Event::WindowEvent { event, .. } => match event {
                WindowEvent::CloseRequested => *control_flow = ControlFlow::Exit,
                _ => {}
            },
```
Once events are cleared run the render graph which actually does the drawing to the window.
```rust
            Event::EventsCleared => {
                factory.maintain(&mut families);
                if let Some(ref mut graph) = graph {
                    graph.run(&mut factory, &mut families, &());
                    frame += 1;
                }

```
Calculate the cpu time this frame took to draw.
```rust
                elapsed = started.elapsed();
                if elapsed >= std::time::Duration::new(5, 0) {
                    *control_flow = ControlFlow::Exit
                }
            }
            _ => {}
        }

```
There has been an exit command so dispose of the graph and record the stats.
```rust
        if *control_flow == ControlFlow::Exit && graph.is_some() {
            let elapsed_ns = elapsed.as_secs() * 1_000_000_000 + elapsed.subsec_nanos() as u64;

            log::info!(
                "Elapsed: {:?}. Frames: {}. FPS: {}",
                elapsed,
                frame,
                frame * 1_000_000_000 / elapsed_ns
            );

            graph.take().unwrap().dispose(&mut factory, &());
        }
    });
}
```

## Main function

The main function can only run if we have dx12 or metal or vulkan enabled as a feature.
```rust
#[cfg(any(feature = "dx12", feature = "metal", feature = "vulkan"))]
fn main() {
```
Create a default config file:
```rust
    let config: Config = Default::default();

```
The winit event loop is used for running the main loop.
This is not neccesary but makes it easier then running our own loop.
```rust
    let event_loop = EventLoop::new();
```
Build a new window with 960 width by 640 height.
```rust
    let window = WindowBuilder::new()
        .with_inner_size((960, 640).into())
        .with_title("Rendy example");

```
Initialize rendy with metal. There is an auto init function but it's not working for me right now.
```rust
    let rendy = AnyWindowedRendy::init(
        rendy::core::EnabledBackend::Metal,
        &config,
        window,
        &event_loop,
    )
    .expect("Failed to create rendy");
```
Create a rendy pipeline with any window using this macro.
```rust
    rendy::with_any_windowed_rendy!((rendy)
        (mut factory, mut families, surface, window) => {
```
This creates the graph builder so we can add in our pipelines later.
```rust
            let mut graph_builder = GraphBuilder::<_, ()>::new();
```
Grab the size from the window.
```rust
            let (width, height) = window.inner_size().to_physical(window.hidpi_factor()).into();

```
Add the triangle pipeline as a node on the graph.
```rust
            graph_builder.add_node(
                TriangleRenderPipeline::builder()
                    .into_subpass()
                    .with_color_surface()
                    .into_pass()
```
The surface is cleared to white.
```rust
                    .with_surface(
                        surface,
                        hal::window::Extent2D {
                            width,
                            height,
                        },
                        Some(hal::command::ClearValue {
                            color: hal::command::ClearColor {
                                float32: [1.0, 1.0, 1.0, 1.0],
                            },
                        }),
                    ),
            );
```
Actually build the graph.
```rust
                let graph = graph_builder
                .build(&mut factory, &mut families, &())
                .unwrap();

```
Now run the whole graph.
```rust
            run(event_loop, factory, families, graph);
        }
    );
}
```

## Vertex Shader
\#S:MODE=vert
```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec3 pos;
layout(location = 1) in vec4 color;
layout(location = 0) out vec4 frag_color;

void main() {
    frag_color = color;
    gl_Position = vec4(pos, 1.0);
}
```
## Fragment Shader
\#S:MODE=frag
```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(early_fragment_tests) in;

layout(location = 0) in vec4 frag_color;
layout(location = 0) out vec4 color;

void main() {
    color = frag_color;
}
```
