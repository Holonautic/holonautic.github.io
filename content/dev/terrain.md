+++
title = "Destructible Terrain"
description = "A guide to fast voxel terrain in standalone VR."
date = 2024-10-23T13:00:00Z
draft = true
template = "dev/page.html"

[taxonomies]
authors = ["Bryn Dickinson"]

[extra]
lead = "Making voxel islands that you can blow up on mobile hardware."
toc=true
math=true
+++

In this article, we'll take a deep dive into how voxel terrain works in <cite>Boom Boom Hamster Doom</cite>. It's the core mechanic of the game, with everything else spinning off from that thing.

I'm going to start with a little theory---nothing too dry, I promise!---and then we'll work our way down to what it takes to make it happen in VR.

## Marching Cubes

### What is a voxel?

Mention voxels and most people will imagine a grid of little cubes, like in <cite>Minecraft</cite>. This is not *exactly* wrong, but just as [a pixel is not a little square](http://alvyray.com/Memos/CG/Microsoft/6_pixel.pdf), a voxel is not a little cube.

Voxels (<b>vo</b>lume <b>el</b>ements) are really just any pieces of data that live on a 3D grid. For our purposes, it's best to think of them as living on the *corners* of a grid, as small points. So if you have a cube, the corners of that cube are eight voxels.

What sort of data can live in a voxel? Pretty much anything. It could have a value that we use to calculate the terrain surface. It could also have some terrain-related property, like how rocky the ground is here, or how badly it's been burned by explosives.

Voxels can be anything, but the general guideline is that voxels should be *small*. In BBHD, a voxel is four bytes. The smaller it is, the easier it is to crunch through it...

### Implicit surfaces

In mathematics, an implicit surface is defined by a 3D function.

Imagine you have a cloud that fills space in some region. Perhaps the inside of a cube...

(cloud cube video)

At any point in space, you can read out a number of how dense the cloud is at that place.

(sampling a point inside the cube)

Now suppose you pick a number - let's say 0.5. If you find all the points where the cloud has the value 0.5, you get a *surface*. On one side of the surface, the cloud only has values above 0.5. On the other side, the cloud only has values that are less than 0.5. The surface neatly divides space into these two regions.

(cube implicit surface video)

This turns out to be a very handy way of representing a surface in a game.

### The road to marching cubes

How is this so? Suppose you have a ball of rock. Inside the ball is solid rock, and outside the ball is empty air. We could store a grid of <i>voxels</i>, which are just one bit: either we're inside the ball or outside the ball.

Why would it be useful to do this? Well, suppose someone comes along and takes a bite out of the ball. If we know which parts of space they've bitten, we can flip all the voxels that have been bitten from one to zero.

However, game engines don't like to draw voxels (usually), they like to draw triangles. So we need to convert our voxels into a mesh of triangles somehow.

The simplest possible marching cubes algorithm works like this. Let's take a cube with a voxel on each corner. Since there are eight voxels, and each one can take two values, there are $2^8=256$ different possible arrangements.

The basic insight of [Marching Cubes](https://en.wikipedia.org/wiki/Marching_cubes) is that this is (barring some weird edge cases) enough information to pin down the surface inside the cube. For each of these possible cubes, you need between zero and four triangles to split the different voxel values, defined by between zero and six vertices. If you do this for *every* cube, you get a surface which divides all the 1s from all the 0s.

The problem with this approach is that the output tends to look very, well, blocky. You get a lot of zigzagging stair-like structures, instead of smooth terrain.

### Smoooooth marching cubes

How can we get something better from marching cubes? Well, imagine if, instead of just voxels storing 0 and 1, they store a number *between* 0 and 1 which is a kind of 'density', just like the cloud we discussed above. But instead of filling all of space, the density only exists on the voxel points.

Now we pick some threshold value, called the <dfn>isolevel</dfn> - let's say 0.5. If the density is less than 0.5, we count it as a zero. And if it's more than 0.5, we count it as a 1.

But, after we've created the triangles, we shift the vertices along the edges of the cube. We can assume that the value of the cloud varies between the values on the eight corners. So we can estimate where this continuous cloud would cross through 0.5, and then shift the triangles we generated along the edges of the cube until they are about where we think 0.5 would be.

For example, let's say on one corner our voxel reads 0.2, and on the other corner, it reads 0.6. The corner of the triangle we generate should be closer to the 0.6 voxel than the 0.1 voxel. So we linearly interpolate the positions of thet wo corners, based on where the isolevel sits in the interval.

With all this in mind, imagine blurring out the values of the ball of rock we discussed before: outside the ball it's a little 'rocky', and inside the ball it's not quite as rocky as it was before. This way, we're effectively storing more information about the shape of the ball. Then, when we use the marching cubes algorithm with interpolation, we get a much cleaner, smoother shape.

For this reason, our voxels store a byte of 'density', somewhere between 0 and 255. This turns out to be enough precision to get smooth-looking islands.

## Creating islands

Our game does not call for spheres, but for islands, icebergs, asteroids and various other natural looking shapes. So we need to generate voxels which will create the sort of shapes we want when they get meshed with marching cubes. It also needs to be very fast, so the players can easily change generation parameters and produce different islands to play on.

We'll talk a little bit later about how, to make it efficient, we need to split the world into small chunks that can be processed in parallel. The upshot of this is that we need to be able to calculate each voxel independently of what's going on elsewhere on the terrain. This rules out some techniques like erosion simulation, and instead limits us to methods based on noise.

The idea is in a sense similar to a shader: we have a 'generation function' which takes as input the 3D position and generates a voxel value. So what should that generation function do?

### Noise

Noise is one of the absolute core tools of procedural graphics, and it refers to a deterministic but random-looking function of space. There are many types: simplex noise, value noise, Voronoi noise etc, which have different results.

Noise functions all generally work by evaluating some kind of spatial hash function that can be computed from a position. You can read many excellent guides on the mathematical details of noise functions on [Catlike Coding](https://catlikecoding.com/unity/tutorials/pseudorandom-noise/); we won't go into too much detail here except to briefly summarise that we primarily used Unity's built in `math.snoise` function to generate simplex noise, and added multiple octaves to create fractal noise.

But noise on its own is generally not enough. We had certain additional requirements: the result needed to look island-shaped, and it needed to generate materials such as rock and sand in a natural-looking way.

### From noise to surface

<cite>Boom Boom Hamster Doom</cite> has two types of map: islands and asteroids.

For islands, we first generate the noise in 2D, and then generate the voxels in vertical columns.

The idea is that as we go up from the bottom of the voxel volume, we subtract the height of our voxel from the value calculated by the noise function. At some point, the decreasing values will pass across the isosurface value, resulting in terrain at that height.

This means the initial generation is essentially a 2D heightmap (for islands), but it can be carved up in a fully 3D way during the game. A slightly more complex approach would be needed to generate things like overhangs or caves.

Much like a fragment shader, which works on groups of four pixels at once, I took the approach of generating four columns of voxels at the same time. This allows me to also calculate the *gradient* of the ground at that position using a numerical method, which can be used as an input to further calculations.

(It also allows certain calculations to potentially more easily be vectorised, although I do not have analytics to prove whether it really is faster.)

### Shaping the island

The noise by itself gives a bumpy surface, looking something vaguely like a mountain range. To make an island instead, we need the height of the land to fall away as we move away from the centre. This led to the idea of 'shape functions': we calculate the noise and multiply it by some other function that gives the 'overall shape' of the island.

What functions are these? Here's what I came up with...

- Gaussian falloff, which creates a nice smooth hill
- smoothstep falloff, which creates a steep cliff around the edge of an island, like a cake - either in circular or rectangular variants
- two Gaussians, creating two nearby islands that blend into each other, or a flat island with a mountain on it
- a Gaussian subtracted from another Gaussian, creating a ring shape (an atoll) or a crescent

Each of these have some randomisation parameters, for example, the height and radius of the Gaussian, or the distance between the two islands in the double Gaussian.

### Shaping asteroids

Asteroids take a slightly different approach. They start with 3D rather than 2D noise, which creates a bunch of lumps in 3D space. We multiply this by one of the 2D generation functions, and also by a vertical Gaussian. This creates a kind of lumpy potato-like shape; the different shaping functions from the island can also be used to, for example, create a hole going through the middle of the asteroid.

## Terrain materials

Our terrain isn't just made of one substance. Depending on the various biomes visited in our game, it might be made of various materials such as rock or ice. Additionally, during the game, the material might get changed---for example, an explosion could burn the terrain.

Each voxel consists of four bytes. The first byte is the density, used to compute the marching-cubes surface. The remaining three bytes store various kinds of material information. Each byte records a sliding scale from 0-255. This allows us to have gradual, more natural-looking transitions between different materials in the shading.

For the first 'mountain' biome, which depicts a rocky island with temperate trees on it, the three bytes are 'rocky', 'sandy' and 'burned'. 'Rocky' captures the difference between soil and rock. 'Sandy' makes that into sand and sandy rock (at least in theory). 'Burned' is not used at generation time, but when explosions damage the terrain, it also increases the burned byte.

### From voxels to vertices

In geometry, a 'vertex' is a point at the corner of a face. In computer graphics, a vertex is essentially a data structure. It stores not just the position of the point, but also the normal (used in shading calculations), the UV coordinate (used in applying textures), and any other data the programmer might care to pass to the GPU, like a vertex colour.

The vertex colour gives us four bytes per vertex (the classic RGBA quadruplet); Unity converts these into floating point values automatically when we use them in the shader. So, when we generate the mesh, we interpolate the rockiness, sandiness and burnedness between the voxels, and store them in the vertex colours. Then, the shader uses those vertex colours to assign the appearance of different materials.

You might notice we have a byte left over there! I still have some plans for that byte, which I'll come back to in a future article.

### Texture sampling

When we display a mesh in computer graphics, we usually apply 'texture' images across the surface to add detail. To calculate which parts of the shader to display, each vertex stores texture coordinates, also known as UVs. This is essentially a pair of floating point numbers.

Assigning UVs is one of the more technical aspects of creating assets for games. You have all sorts of considerations---texel density, distortion, and seams to name a few. But since we are procedurally generating meshes from voxels, we have no way to assign UVs. A different approach is needed.

We could sample the texture simply from the vertex position, but the problem is textures are 2D and the world is 3D, so there will always be a direction where this texture appears stretched. Let's call this the distorted direction.

The solution to this is a method called 'triplanar mapping'.

Triplanar mapping is useful for textures where there is a lot of even detail, like a grass or dirt texture. The insight of this method is that the stretching will be most problematic on faces that are parallel to the distorted direction. So, if we sample the texture three times, each at right angles to the others, we can pretty much cover our bases; then each pixel can choose which texture to use that will give the least stretching.

We use the surface normal to work out which way a surface is facing. If the normal is lined up perfectly with a given texture, we just use that texture. If it's in-between, we can blend between the two textures. As long as the textures have lots of small details, any artefacts from the blending are hard to see.

## Implementing marching cubes terrain

We are not the first people to make Marching Cubes. Our method builds off the work of [Eldemarkki](https://github.com/Eldemarkki/Marching-Cubes-Terrain), although in the end I rewrote most of his code. I was in part trying to optimise it, in part adjusting it to fit our needs, but honestly the biggest gain is that rewriting something teaches you a *lot* about how it works.

Eldemarkki's code already makes use of Burst-compiled jobs, which allow us to move the voxel calculations off the main thread. Since the Quest is not fast enough to process the entire island's generation and marching cubes in a single frame, we use an approach that allows us to spread the work over multiple frames.

### Chunks

Like most voxel based games, we don't store all our voxels in one giant array, but split them into 'chunks' that cover a cuboid volume of the game world.

Each chunk is associated with an array of voxels, and generates its own mesh. It overlaps with the adjacent chunk at the edge (there are some redundant voxels at the edges of each chunk), so the meshes line up perfectly, even when they get blown up. That's the basic idea.

In code, each chunk is associated with a `NativeArray` of voxel data structs... well, sorta.

```cs
public struct VoxelDataAtlas : IComponentData
{
    public NativeParallelHashMap<int3, UnsafeArray<VoxelDataElement>> chunks;
}
```
Unity's safety system gets fussy if you try to nest native containers inside a job. Since we knew our access pattern would be safe, we made an 'unsafe' variant of a `NativeArray` which does not have these restrictions.

There is evidently some indirection here. To get your hands on voxel data, you have to have the `int3` index of the chunk, which gets hashed to look up an `UnsafeArray` struct, which controls the pointer to the actual array. You only really need to do this a few times though, so it's not a big issue.

### The dance of the jobs

So how do we actually process that data? As mentioned above, our tools are two Unity features called the Burst compiler and Job system.

Burst compiles a limited subset of C# using LLVM-based tech to get performance comparable to an optimised C compiler; it is kind of like a limited version of Rust, a low-level language with some features from a more recent one. The main limitation is that it rules out any managed code: no non-static classes, only struct; only blittable value types; and so on. And, most importantly, you have to manage your own memory using Unity's 'Native' data structures (`NativeArray`, `NativeHashMap` and so on), which contain a pointer to the actual data and some kind of thread safety information that interacts with the job system.

The job system is designed to make it easy to have your game code be multithreaded. You create structs which implement an appropriate interface, such as `IJob`, and hand them over to Unity to schedule on one of a number of worker threads, depending on the cores available on your CPU. On the Quest, we only have two worker threads. Job scheduling involves a significant overhead, a subject we'll get into in another article, so for small jobs it's frequently better to do them on the main thread, but these aren't small jobs.

For each chunk, a series of jobs need to be performed at terrain generation time:

- first, a voxel generation job must generate the initial voxel values
- second, a marching cubes job must generate the mesh data (vertices and triangles)
- third, a physics collider job must generate a collider from this mesh data

When all the chunk meshes are generated, we can combine and smooth the meshes to create the render mesh---more on that later.

Now, whenever something modifies the terrain, such as an explosion, we modify the voxel data for certain chunks. Then we need to re-run this process for each modified chunk, starting from the marching cubes stage.

We can think of these jobs like links in a chain. On each frame, we check the active jobs to see if they're finished yet. If they are, we extract the finished data, deallocate any arrays that need deallocating, and schedule the next job in the chain if necessary. The algorithm goes something like...

- check if any collider jobs have finished, and update the entities
- check if any marching cubes jobs have finished, and schedule collider jobs for each one
- check if any voxel generation jobs have finished, and schedule marching cubes jobs for each one
- schedule voxel generation jobs up to the current limit
- if no marching cubes jobs are pending, combine and smooth the meshes

Since it has to deal with managed types like meshes, the terrain system has to be a managed system. The pending jobs are recorded in dictionaries which map the chunk index (see below) to the corresponding pending job. Additional dictionaries keep track of the entities for each chunk and the meshes for each entity. This is not necessarily the most efficient storage, but it is not the bottleneck.

With this high-level overview in hand, lets drill into the details.

## The coordinate system forest

One of the headaches I encountered when I first started on this project was the rather inconsistent language used to refer to the different coordinates - voxels and cubes, coordinates or indices, or maybe even an 'Xyz'. Not to mention needing Axis-Aligned Bounding Boxes for certain calculations.

I wanted consistent language to make reasoning about everything else easier. The language I settled on is...

<dl>
<dt>position</dt>
<dd><code>float3</code>, position vector relative to the origin of the voxel world</dd>

<dt>chunk index</dt>
<dd><code>int3</code>, locates a chunk in the world</dd>

<dt>voxel index</dt>
<dd><code>int3</code>, locates a voxel inside a chunk</dd>

<dt>linear voxel index</dt>
<dd><code>int</code>, the actual array index of a voxel</dd>

<dt>voxels per chunk</dt>
<dd>positive `int3`, the number of voxels along each axis of a chunk</dd>

<dt>cubes per chunk</dt>
<dd>one less on each axis than the <i>voxels per chunk</i>, since cubes sit in between voxels</dd>

<dt>dimensions</dt>
<dd>the floating point size of something (e.g. a chunk or the whole world)</dd>

<dt>bounds</dt>
<dt>AABB</dt>
<dd>the standard floating-point axis-aligned bounding box provided by Unity</dd>

<dt>voxel AABB</dt>
<dd>a structure that describes a cubic region of voxels inside a chunk, with <i>voxel indices</i> for the corners</dd>

</dl>

This is why I called the lookup for voxel data an 'atlas' rather than the more obvious 'index'. <small>There's a reason naming is one of the hard problems of computer science, let's just say.</small>

### Finding your place

With this scheme in mind, we needed code to map from positions in regular floating-point space to the discretised voxel space, and back. To add an additional wrinkle, I wanted the number of chunks along a given axis to potentially be an odd number, and still have its centre on the origin. In that case, the chunk at (0,0,0) is centred on the origin; otherwise it has its bottom-left-back corner on the origin.

This led to a bunch of functions like this...

```cs
public int3 chunkIndexContaining(float3 position)
{
    float3 axisOddnessOffset = 0.5f * (float3) (chunksPerWorld % 2);

    return (int3) math.floor(((position - origin) / chunkDimensions) + axisOddnessOffset);
}
```
...which takes a position in world space and figures out which chunk contains it. Nothing too complicated, but making sure there aren't any sign errors can be fiddly.

### A brief aside on data ordering

'3D' arrays are essentially a bunch of concated 1D arrays, so there is also a question of which order to arrange the voxel data in a chunk. You want data to be easily read into the CPU as full cache lines, to minimise the amount the CPU (fast) has to talk to memory (slow). For this reason, I wanted the y direction to be the most closely packed; the conversion from voxel index to linear voxel index is...

```cs
public int linearVoxelIndex(int3 voxelIndex)
{
    return voxelIndex.z * voxelsPerChunk.y * voxelsPerChunk.x + voxelIndex.x * voxelsPerChunk.y + voxelIndex.y;
}
```
Crucially this is *voxels* per chunk not *cubes* per chunk.

I have not actually measured whether this ordering actually makes generation faster. Maybe it's a pointless non-optimisation. However, it does keep things conceptually simple during generation.

## Voxel generation: the nitty gritty

So with all the preliminaries out of the way, this is the *interesting* part. In my book, anyway.

As discussed above, the basic calculation is to generate a 2D heightmap using fractal gradient noise (based on the `snoise` function provided by Unity), multiply it with a shaping function, and then generate the voxel values by decreasing linearly in columns along the y direction. To this basic calculation, I added some additional features like gradients and terracing.

The general approach at the generation stage is to provide myself with useful pieces of information that I can combine artistically. For example...

### Gradients

I generate four columns of voxel data at once, essentially a little square, of coordinates along the lines of (x,y), (x+1, y), (x, y+1) and (x+1, y+1). This allows me to write many steps in terms of vector types like `float4`. At the time I believed this would hopefully allow Burst to more readily use SIMD commands to crunch along the data faster, but I hesitate to claim that is true without profiling and looking at the actual compiled code.

The more tangible advantage of processing vertices in this way is that we can compute the gradient of the terrain numerically.

First we generate the floating-point position of each voxel column:

```cs
float2 voxelPositionNormalised00
    = (chunkBounds.Min.xz + new float2 (x, z) * voxelSpacing - worldBounds.Center.xz) / worldBounds.Extents.xz;
float2 voxelPositionNormalised10
    = voxelPositionNormalised00 + new float2(voxelSpacing, 0) / worldBounds.Extents.xz;
float2 voxelPositionNormalised01
    = voxelPositionNormalised00 + new float2(0, voxelSpacing) / worldBounds.Extents.xz;
float2 voxelPositionNormalised11
    = voxelPositionNormalised00 + new float2(voxelSpacing, voxelSpacing) / worldBounds.Extents.xz;
```

Then we feed each of these into a noise function and evaluate the shape function at each position:

```cs
float4 terrainNoise
    = new float4 (
        OctaveNoise(voxelPositionNormalised00, terrainSettings.noiseFrequency,
            terrainSettings.noiseOctaveCount, noiseOffset) * terrainSettings.amplitude,
        OctaveNoise(voxelPositionNormalised10, terrainSettings.noiseFrequency,
            terrainSettings.noiseOctaveCount, noiseOffset) * terrainSettings.amplitude,
        OctaveNoise(voxelPositionNormalised01, terrainSettings.noiseFrequency,
            terrainSettings.noiseOctaveCount, noiseOffset) * terrainSettings.amplitude,
        OctaveNoise(voxelPositionNormalised11, terrainSettings.noiseFrequency,
            terrainSettings.noiseOctaveCount, noiseOffset) * terrainSettings.amplitude
    );

terrainNoise
    *= new float4 (
        falloffFunction.falloff(voxelPositionNormalised00, random),
        falloffFunction.falloff(voxelPositionNormalised10, random),
        falloffFunction.falloff(voxelPositionNormalised01, random),
        falloffFunction.falloff(voxelPositionNormalised11, random)
    );
```
We now have the noise values for all four columns in a float4.

I calculate the gradient in a way that is similar to the `ddx`, `ddy` and `ddxy` functions [available in shader programming](https://gamedev.stackexchange.com/questions/62648/what-does-ddx-hlsl-actually-do). I method called the [Roberts cross](https://en.wikipedia.org/wiki/Roberts_cross), due to technicalities discussed in [this blog post by Bart Wronski](https://bartwronski.com/2021/02/28/computing-gradients-on-grids-forward-central-and-diagonal-differences/)...

```cs
float terrainDerivative = math.length(new float2 (terrainNoise.w - terrainNoise.x, terrainNoise.y - terrainNoise.z));
```
This generates a 2D vector representing the magnitude of the gradient at this point. <small>Strictly speaking, I should additionally multiply this by $\sqrt(2)$, but since I always multiply it by some arbitrary parameter before using it, that can be skipped.</small>

Why do we want to know the gradient? Well, we can use it for various subsequent calculations. For example, I wanted steep slopes to show exposed rock faces. I can reduce the thickness of the sediment layer based on the gradient. For now, that's all I'm using gradients for.

### Terracing

One of our levels is an ice level, involving essentially a large floating iceberg. One of the interesting properties of ice is that it tends to form large flat slabs, rather than the smoothly varying bumps of our noise. So, I came up with the idea of adding a 'terracing' effect to the heightmap, pushing towards creating a 'stairs'-like effect.

The core idea of 'terracing' is that by adding an appropriately scaled sinusoid to the raw output of the generation function, you can push a linear slope towards something a bit more like steps. The calculation is $$t(x)=x+ap\sin\frac{x}{p}$$where $p$ is the period of the steps and $a$ between 0 and 1 controls how strong the effect is. (You can potentially iterate this calculation multiple times to get even steeper steps.)



## Draw calls and smoothing

When I joined the project, every chunk generated its own little mesh. Since we're rendering in Entities Graphics, it is *supposed* to do clever batching stuff in the way it sends data to the GPU. This didn't seem to be happening; instead we had some 50-something draw calls simply from terrain chunks.

Draw calls are very slow on the Quest. One of the first big performance wins was to combine all the generated terrain meshes into one single render mesh and only render this, meaning only one draw call for the whole terrain. I don't remember exactly how many milliseconds this saved, but it went from sub-72fps to clean 72fps.

But combining the meshes in this way gave another, even more exciting possibility. The marching cubes algorithm, by default, generates a sharply faceted triangle mesh. Each cube generates its own vertices, with their own normals. To get our terrain to look smoother, we would need to **combine vertices at the same location**, and **average out their normals**.

The problem with combining vertices based on position is that two vertices in the 'same' position might have an imperceptible difference thanks to the imprecision of floating point calculations. So you need to consider vertices that are 'close enough' to be the same. The solution to this is actually remarkably simple...

- multiply every position by some fixed constant (the larger the constant, the smaller the maximum gap between vertices to be combined)
- round each coordinate to an integer (so you've converted a `float3` to an `int3`)
- use a hashmap (i.e. a dictionary) to map each integer position to a linear index, which we will call the <dfn>canonical index</dfn>
- every time you test a vertex, check if that position already exists in the hashmap. if it does, use the existing canonical index. if not, add the vertex data (*not* rounded) to the canonical vertices array and store its index in the hashmap
- either way, add the vertex normal to the normal of the canonical vertex
- store another array with each the canonical index for each original index

Then once you're done, you iterate over the canonical vertices again and normalise their normals (meaning, you divide each vector by its length, making it a valid normal vector). You also need to update the triangles, which are indices into the vertex array, to point to the *new* indices into the vertex array.

Here's the code which does that, which is perhaps easier to understand:

<details>
<summary>Code for smoothing vertices</summary>

```cs
[BurstCompile]
public struct CombineAndSmoothVerticesJob : IJob
{
    [ReadOnly]
    public NativeArray<MeshingVertexData> vertexData;

    [ReadOnly]
    public NativeArray<uint> indices;

    public NativeArray<uint> newIndices;

    public NativeArray<uint> indexRemapping;

    public NativeList<MeshingVertexData> canonicalVertexData;

    public NativeHashMap<int3,uint> canonicalIndices;

    [BurstCompile]
    public void Execute()
    {
        uint canonicalVertices = 0;

        //find all unique vertices
        for (int i = 0; i < vertexData.Length; ++i)
        {
            uint canonicalIndex;
            if (!canonicalIndices.TryGetValue(Key(vertexData[i].position),out canonicalIndex))
            {
                canonicalIndex = canonicalVertices;
                ++canonicalVertices;
                canonicalIndices.Add(Key(vertexData[i].position), canonicalIndex);
                canonicalVertexData.Add(vertexData[i]);
            } else {
                var vertex = canonicalVertexData[(int) canonicalIndex];
                vertex.normal += vertexData[i].normal;
                canonicalVertexData[(int) canonicalIndex] = vertex;
            }
            indexRemapping[i] = canonicalIndex;
        }

        //remap the index buffer. the length should stay the same
        for (int i = 0; i < indices.Length; ++i)
        {
            newIndices[i] = indexRemapping[(int) indices[i]];
        }

        //normalize the summed vertices
        for (int i = 0; i < canonicalVertices; ++i)
        {
            var vertex = canonicalVertexData[i];
            vertex.normal = math.normalize(vertex.normal);
            canonicalVertexData[i] = vertex;
        }
    }

    [BurstCompile]
    private int3 Key(float3 position)
    {
        //more precision = smaller distance to merge vertices
        //e.g. with precision 100 vertices must match to the 2nd decimal place to merge
        const int precision = 100000;
        return (int3) math.round(position * precision);
    }
}
```
</details>

I don't average the other vertex data such as colour, since this logically *should* all be consistent for vertices that are at the same location.

On the device this generally takes less than two milliseconds running on the main thread, so it can easily be done within a single frame as soon as the marching cubes jobs are finished. It's actually comparable to the time that Unity takes simply to combine the meshes prior to smoothing, and we might still be able to save a bit of time by concatenating the arrays ourselves inside Bursted code, at the cost of some complication in managing the index buffer.

