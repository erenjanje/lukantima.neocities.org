# CENG795-Final: Ghost in the Trace

[^1] [^2] [^3]

- [Theory](#theory)
- [Implementation](#implementation)
  - [Sparse Voxel Octree](#sparse-voxel-octree)
    - [Types](#types)
    - [Input Augmentation](#input-augmentation)
    - [Sparse Voxel Octree Creation](#sparse-voxel-octree-creation)
    - [Sparse Voxel Octree Traversal](#sparse-voxel-octree-traversal)
    - [Texture Support](#texture-support)
  - [Minecraft World](#minecraft-world)
    - [World Format](#world-format)
    - [Materials \& Textures](#materials--textures)
- [Results](#results)
- [Performance](#performance)
- [Conclusion](#conclusion)

Kon'nichiwa and ciao everybody, welcome to our blog post about the final project of the course Advanced Ray Tracing. It has been a very interesting, challenging, educating and entertaining semester with ray tracing. After countless hours, we have managed to create (mostly) working ray tracers with a great set of features. As we have finished the homeworks, we decided our final projects, each of us choosing a different augment. Alp and me decided to implement Sparse Voxel Octree, which is a nice acceleration structure that can be used to represent huge amounts of blocks while taking small amount of space, and also have a fast intersection routine.

I (Erencan) did the SVO part while Alp did the Minecraft importer part. The project is based on my ray tracer, so its rendering has all the quirks and errors that my HW6 ray tracer has.

## Theory

Voxel Trees work in a way pretty similar to Bounding Volume Hierarchies. Actually, if every block is defined as 12 triangles, which is pretty much the normal representation of a cube, and the coordinates of the vertices of these triangles are given correctly (without precision errors), a BVH pretty much converges to a voxel tree. If each BVH node has eight children, this BVH tree is pretty much a Dense Voxel Octree. I am not sure whether surface area heuristic can break this or not, but median division (which is what both of us used) converges to a DVO.

DVOs are pretty bad that they need more memory compared to a plain block array. They have a nice property of that they can easily be represented using heaps, since each internal node has exactly 8 children. The memory usage can be calculated as a geometric series:

$$
M = 1 + 8 + 8^2 + 8^3 + ... + 8^{\log_8\left(N\right)} = \sum_{i=0}^{\log\left(N\right)} 8^i
$$

where $N$ is the number of blocks. For a geometric series with base $\alpha$ from $1$ (exponent of $0$) to $n$, the summation can be calculated using this formula:

$$
\frac{\alpha^{n+1}- 1}{\alpha - 1}
$$

Here, $n = \log_8\left(N\right)$ and $\alpha = 8$ for our case. When we put them into their places in the summation formula, we get this:

$$
\frac{8^{\log_8\left(N\right)} - 1}{7}
$$

The $8$ and $\log_8$ eliminates each other, resulting in

$$
\frac{8N - 1}{7}
$$

Since $8N$ dominates the upper term, we get $M = \frac{8}{7}N$, the total memory consumption of a DVO for $N$ blocks. Even though the overhead is only $\frac{1}{7}N$, since $N$ can be _very_ large, this is a problem. For example, Minecraft worlds have 16x256x16 (or 16x320x16 in newer versions) chunks, which can be considered as _tiny_ compared to a regular Minecraft world a player plays on, and such a chunk has 4096 blocks. Far render distance enables rendering of 16 chunks in each horizontal distance (since Minecraft chunks are not stacked on top of each other without modifications), which means 1024 chunks, which means more than 2 millions of blocks! As you can see, it is inconvenient to store every block.

Now think about a regular Minecraft world. Most of the world's upper half is actually empty, since the sea level is `y=62`, so a non-mountainous region has pretty much three quarters of the chunk as empty. On top of that, most of the lower parts are actually huge blobs of stones, with ores and caves being occasional. This is a _huge_ lack of entropy! Sparse Voxel Octrees exploit this kind regularity by _merging_ the nodes of a DVO. Unlike DVOs, SVOs cannot be represented using heaps, they are more similar to BVH in this regard. However, since they exploit these regularities, they are much more compact, which improves efficiency by not only improving cache efficiency, but also making it possible to terminate a traversal before a full dive into the tree. In DVOs, every intersection check goes to the deepest level of the tree; however, in SVOs, a check can terminate if it comes across a bigger block group. So, let's have a look at its implementation.

## Implementation

### Sparse Voxel Octree

#### Types

The firt thing I did was deciding on the representation of the tree nodes. In principle, they are simple sum types of materials and internal nodes. However, I struggled a bit about how to represent them. A separate boolean would be a huge waste, since a boolean uses up 1 or 4 bytes, not a single bit. I thought about using bitfields, but I scrapped this idea in favor of the same approach I used for BVH: using negative and positive integers. Unlike BVH, negative indices this time represent internal nodes while non-negative ones represent materials. This is actually pretty nice that air blocks are represented using material 0, which is considered as a material (i.e. a leaf node) instead of an internal node. I created a few helper functions to make handling the index easier.

#### Input Augmentation

The next thing I did was defining the format of the input. The XML format we have been using until now did not have a way to define blocks. For this, I defined two new tags:

1. A `Chunk`: A chunk represents a single instance of SVO. It is defined under `Objects`. Like other objects, it has an `id`, which may be non-contiguous but as the other objects, but unlike meshes, `id`s are ignored. This should not cause any issues since terrains do not currently support instances. Supporting instancing could have been nice, but rendering bigger worlds was our priority, and we had a small timeframe after HW6 and before deadline. A `Chunk` also supports attribute `scale`. `scale` is defined as a power of 2, so `0` means `1`, `1` means `2` and `-1` means `0.5`. This scale is the size of this chunk. Blocks defined out of this size cause errors, but increasing this also decreases performance since this is pretty much the starting "height" of the tree. A `Block` with scale `n` defined under a `Chunk` with scale `m` has a **maximal** depth of `m-n`.
2. A `Block` is defined under a `Chunk`. It represents a single block in this chunk. It is a short tag (ends with `/>`) since a block does not have any child. It is defined by its attributes.
   1. `x`, `y` and `z` define the position of the block. Note that these coordinates _must_ be aligned with the size of the block. For instance, a block with `scale` `0` (i.e. a 1x1x1 block) cannot have `x` `0.5`, but a block with `scale` `-1` (i.e. a 0.5x0.5x0.5 block) can have such a coordinate.
   2. `material` defines the material of the block. It is pretty much the same as the `Material` element inside meshes, spheres, triangles and instances in the homeworks, with the only difference that it is an attribute rather than an element. I is also 1-based.
   3. `scale` defines the size of the block as a power of 2. It is pretty much the same as the `scale` of a `Chunk`, only that it is for a block. As said in the coordinate article, `scale` and coordinates must result in an aligned block.

Apart from this, there is an improvements to another part of the format. Namely, the materials now support textures, since defining textures separately for each block would be bad, and in most scenes in real life including a Minecraft scene, it is materials that utilize the textures. This would be an interesting discussion, since a texture can define both color (diffuse and specular) _and_ geometry, so a texture may belong to either of them, philosophically speaking. We went with this since defining both textures and materials per block would be pretty wasteful, especially since _every_ SVO node would need to allocate area for both of them. 

With this, the input was done. The next step was obvious: creating the SVO!

#### Sparse Voxel Octree Creation

After kilometers of walking inside the dormitories and the home, I realized that SVO creation was pretty much the same as BVH. After reading all the blocks in a `Chunk`, `Chunk::construct` method is called. This separation is nice since Alp's importer also gives a block array, so the same constructor can be used to construct both imported Minecraft worlds and hand-defined chunks.

I can actually just post the construction method, since it is pretty short, yet does most of the work.

Actually, most of the function is just the same code repeated eight times.

The main algorithm is as follows:

1. For a range of blocks, check whether it should be divided further or not.
   1. An empty range means there is no block belonging to this node, so we return empty node
   2. If there is only one block and its size matches the size of the node (i.e. their scales are equal), this is a leaf node with material defined as the material of that block.
   3. Otherwise, if any of the blocks in the range has size larger than the size of the current node, something is wrong. It may be caused by two reasons, both of which would cause infinite loop in construction if this check does not exist:
      1. There are overlapping blocks. This is actually the reason I added this check for. If a block's size is equal to the size of the node while there are other blocks that belong to the node, this means that the bigger block overlaps with the other ones. This causes infinite loop since after this point, size gets only smaller than the size of this block, so no matter how deeper the function divides the blocks, at least one recursion will not terminate.
      2. There are _multiple_ (or some _single_) out-of-range blocks. I did not anticipate for this problem, but after Alp's tests, we found out that out-of-range blocks also make this predicate hold true. This is due to the fact that all out-of-range blocks are always grouped into the same "octant" no matter the depth, so at some point, the size of the node becomes smaller than the size of at least one of these blocks. A single out-of-range block actually does not cause infinite loop in some cases, since at some point, either it causes this error if it is on the same octant with another in-range block (since both of them will be put into the same octant as the size gets smaller) or it becomes the sole block of the octant, making the node's material its own. This is actually a pretty significant error since it changes the position of this out-of-range block. However, most cases would actually be handled by the predicate, and this degenerate case is not a big problem, since a legal scene should not have it.
2. If neither of these conditions hold true, this node is an internal node.
3. Partite the range so that it has 8 (possibly empty) partitions each representing an octant. This is done by comparing the coordinates of the block with the coordinates of the center of _node_. This center of node has a pretty simple calculation: each octant is offset by a quarter of the size of the node. For instance, if the node has size $256$ with center $(0,0,0)$, the center of the first octant (all coordinates are non-negative) is $(64,64,64)$.
4. Recursively construct each child with their corresponding ranges while decreasing the size.
5. If all children have the same material, then collapse them into the same node. Otherwise, add these children to the array.

The collapse part was actually pretty interesting, since I stumbled upon almost the same problem in my compilers course! Specifically, constant folding. Say we have an expression `3 + 5`, which is parsed into a node representing addition operator, its left child being a constant `3` and the right child being a constant `5`. In constant folding pass in a compiler, this addition node must check whether all of its children are compile time constants, do the operation directly and change itself to a compile time constant with the result obtained from this operation. However, unlike this, implementing it was pretty harder since the nodes were polymorphic objects, so what I needed was pretty much changing the type of `this`! I solved the issue by delegating this change to the parent node, which made the code unnecessarily more complex. The `SvoNode`s are pretty much ad-hoc sum types, so this approach can easily work with sum types but not polymorphic objects. A nice lesson to have in the great adventure of programming!

When I converted this to code, the result was this:

```cpp
const auto length = end - start;
auto erroneous_scale = 0;
if (length == 0) {  // Empty
    makeMaterial(0);
    return;
} else if (length == 1 and scale == chunk.blocks[start].scale) {  // Node equals a whole block
    makeMaterial(chunk.blocks[start].material);
    return;
} else if (std::any_of(chunk.blocks.cbegin() + start, chunk.blocks.cbegin() + end, [&](Block const& block) {
                if (block.scale >= scale) {
                    erroneous_scale = block.scale;
                }
                return block.scale >= scale;
            })) {
    throw std::runtime_error("Overlapping blocks detected");
}

// Node is an internal node

const auto offsets = SvoNode::getOffsets(offset, scale);

const auto octants = chunk.partition(start, end, offset);

const auto [start_0, end_0] = octants.xyz;
const auto [start_1, end_1] = octants.xyZ;
const auto [start_2, end_2] = octants.xYz;
const auto [start_3, end_3] = octants.xYZ;
const auto [start_4, end_4] = octants.Xyz;
const auto [start_5, end_5] = octants.XyZ;
const auto [start_6, end_6] = octants.XYz;
const auto [start_7, end_7] = octants.XYZ;

auto children = SvoChildren();

children[0].construct(chunk, offsets[0], start_0, end_0, scale - 1);
children[1].construct(chunk, offsets[1], start_1, end_1, scale - 1);
children[2].construct(chunk, offsets[2], start_2, end_2, scale - 1);
children[3].construct(chunk, offsets[3], start_3, end_3, scale - 1);
children[4].construct(chunk, offsets[4], start_4, end_4, scale - 1);
children[5].construct(chunk, offsets[5], start_5, end_5, scale - 1);
children[6].construct(chunk, offsets[6], start_6, end_6, scale - 1);
children[7].construct(chunk, offsets[7], start_7, end_7, scale - 1);

auto material = int32_t(-1);
bool is_filled = true;
for (const auto& node : children) {
    if (not node.isMaterial()) {
        is_filled = false;
        break;
    }

    if (material == -1) {
        material = node.getMaterial();
    }

    if (material != node.getMaterial()) {
        is_filled = false;
        break;
    }
}

if (is_filled) {
    makeMaterial(material);
} else {
    makeInternal(chunk.nodes.size());
    chunk.nodes.emplace_back(std::move(children));
}
```

`makeMaterial` and `makeInternal` changes the object itself, which I explained in the [types](#types) section. The layout of octants is a bit interesting. The field names are all `xyz`s but the capitalization denotes the sign. If that coordinate is negative, it is lowercase, otherwise uppercase. So `xYz` denotes the octant where `x` and `z` are negative and `y` is non-negative. I sorted them so that if lowercase ones are converted to `0`s and uppercase ones are converted to `1`s, the order reads like binary counting. `SvoChildren` is a simple `using`, it is just an `std::array` of 8 `SvoNode`s.

I actually struggled a bit while trying this function. In previous iterations, I pushed/emplaced the nodes directly to the nodes array and called the `construct` method on them. Did you see the problem again? This would invalidate all the nodes if the array ever reallocates, including the parents! Yikes (again). Almost one third of my time was spent on this single issue. The funny thing is that a similar issue would have happened with BVH nodes, but instead of using references (including `this`), I used indices. It seems like past me was definitely smarter than now me.

So, the construction is finished!

#### Sparse Voxel Octree Traversal

If SVO creation made me use my brain by an amount of $x$, I think traversal made me use it around $\frac{x}{5}$. It was similar to BVH traversal, but it was even easier, probably since intersecting a material node directly means intersection, whereas in BVH I would need to call the callback ([see the relevant part of HW2 post](erencanc.pages.dev/posts/2024/11/ceng795-hw2#bounding-volume-hierarchy)). Another change I did was adding `t` support to `aabbIntersection`. In BVH, `t` is not needed since the nodes are only used for true/false checks; however, in SVO it is used since the intersection itself is done with the bounding box itself. Fortunately, the `aabbIntersect` calculates `t`, it just ignores it after return. I just added a new optional `float*` parameter, and put the found `t` if it is not `nullptr`. The intersection algorithm is even shorter than creation:

1. If the block is empty/air, skip it (there cannot be intersection in this case).
2. If there is not intersection, return `false`.
3. If there is intersection, there are two possibilities:
   1. If it is a leaf node, save the material stored in the node since the intersection means an intersection with the object (block) itself and return `true`.
   2. It it is an internal node, recursively try every child and return `true` if _any_ of these children has intersection.

The code is longer, but mostly due to repeated recursive calls.

```cpp
if (node.isMaterial() and node.getMaterial() == 0) {
    return false;
}

let bb = BoundingBox::getBoundingBoxOfSvoNode(offset, scale);
var t_candidate = 0.0f;
if (aabbIntersection(*this, bb, &t_candidate)) {
    if (node.isMaterial()) {
        if (t_candidate < t_out) {
            t_out = t_candidate;
            material = node.getMaterial() - 1;
            return true;
        }
    } else /* if(node.isInternal()) */ {
        let& children = chunk.nodes[node.getInternal()];
        let offsets = SvoNode::getOffsets(offset, scale);

        let child_0 = intersect(chunk, children[0], offsets[0], scale - 1, t_out, uv, material, n);
        let child_1 = intersect(chunk, children[1], offsets[1], scale - 1, t_out, uv, material, n);
        let child_2 = intersect(chunk, children[2], offsets[2], scale - 1, t_out, uv, material, n);
        let child_3 = intersect(chunk, children[3], offsets[3], scale - 1, t_out, uv, material, n);
        let child_4 = intersect(chunk, children[4], offsets[4], scale - 1, t_out, uv, material, n);
        let child_5 = intersect(chunk, children[5], offsets[5], scale - 1, t_out, uv, material, n);
        let child_6 = intersect(chunk, children[6], offsets[6], scale - 1, t_out, uv, material, n);
        let child_7 = intersect(chunk, children[7], offsets[7], scale - 1, t_out, uv, material, n);
        return child_0 or child_1 or child_2 or child_3 or child_4 or child_5 or child_6 or child_7;
    }
    return false;
}
return false;
```

So, let's test it! I added this small block to the scene:

```xml
<Chunk id="1">
    <Block x="0" y="-1" z="0" material="1" />
</Chunk>
```

which results in this from a nice angle:

![Simple block test](./images/simple_block_no_normal.png)

Success? Success!

![Yep, a meme](./images/But_It's_Honest_Work.jpg)

Of course, the project was not exactly done. Even though simple materials were supported, the UV coordinates were not. This was a serious problem since Minecraft blocks are not defined as colors, but textures. Furthermore, there is some noise in sides of the cube as you can see, the reason is that the normal is always returned as $(0,1,0)$! I implemented the normal and UV coordinate support at the same time, since they used pretty much the solution of the same problem: face detection.

#### Texture Support

The first thing to add in order to support textures was UV support, since textures cannot be used without texture coordinates. In order to do this, I actually did a small hack. Since no matter the transform of the chunk, the hit points would be local, so the normals would _always_ be parallel to the base axes. The `aabbIntersect` does not find the face, so I needed to find it. After some thinking, I realized that I could just check the sides of the bounding box with the local hit point. It would be pretty enough to find the face. First, I wrote this function:

```cpp
static glm::vec3 getBlockNormal(glm::vec3 hit_point, BoundingBox aabb, glm::vec2& uv) {
    var i = glm::vec2();
    if (equalf(hit_point.x, aabb.start.x)) {
        uv = modf(glm::vec2(hit_point.z, -hit_point.y));
        return {-1, 0, 0};
    }
    if (equalf(hit_point.y, aabb.start.y)) {
        uv = modf(glm::vec2(hit_point.x, -hit_point.z));
        return {0, -1, 0};
    }
    if (equalf(hit_point.z, aabb.start.z)) {
        uv = modf(glm::vec2(-hit_point.x, -hit_point.y));
        return {0, 0, -1};
    }

    if (equalf(hit_point.x, aabb.end.x)) {
        uv = modf(glm::vec2(-hit_point.z, -hit_point.y));
        return {1, 0, 0};
    }
    if (equalf(hit_point.y, aabb.end.y)) {
        uv = modf(glm::vec2(hit_point.x, hit_point.z));
        return {0, 1, 0};
    }
    if (equalf(hit_point.z, aabb.end.z)) {
        uv = modf(glm::vec2(hit_point.x, -hit_point.y));
        return {0, 0, 1};
    }
    return {0, 0, 1};
}
```

where `equalf` is just float equation but with an epsilon, and `modf` is `glm::modf` but done twice to remove the sign. `std::` and `glm::modf` both return negative numbers if the input is negative, which is not wanted, so I applied the `modf`, added `1.0f` and took `modf` again. This is pretty much the `MODULUS` macro I used in Perlin textures (defined as `#define MODULUS(a, b) (((a % b) + b) % b)`), but for floats.

The signs in the UV were determined by hand. For sides, I tried to do it so that when looked directly, the top left is $(0,0)$, the U component increases to the right and V component increases to the down, which is pretty much the same as how an image is read. For top and bottom, I was not sure, so it was pretty much random.

We can render the scene using only UV coordinates to see the result:

![Simple block with UV](./images/simple_block_uv.png)

We can also render with shading to see the effect of the normal:

![Simple block with shading](./images/simple_block_shaded.png)

The last thing I did was adding texture support to materials. After struggling with the pointers a bit due to the architecture of the program, I managed to do it by using `std::optional`, which is a specific sum type implementation in C++ STL. After this, I rendered using bricks:

![Simple block with texture](./images/simple_block_texture.png)

Phew, I really thought that I would stumble upon weird issues like the one I came accross in construction and the time would not be enough to finish the project. However, it was finished!

I also tried with smaller and bigger blocks. It should not be an issue, but I tried anyways, and it was not problematic as I expected.

![Smaller block with texture](./images/simple_block_small.png)

![Bigger block with texture](./images/simple_block_big.png)

The result was pretty nice, but I assume that one of the biggest reasons is that the texture was seamless. Let's render the UVs.

![Smaller block with UV](./images/simple_block_small_uv.png)

![Bigger block with UV](./images/simple_block_big_uv.png)

Big block looks pretty good, it really looks like it consists of smaller full blocks. However, the smaller block looks a bit off. The problem is that we have not been able to test it throughly.

So, my work was done. Let's hear Alp's work in his own words. See you in results and conclusion (but not performance)!

### Minecraft World

We were in dire need of some things to render other than single blocks. We decided to use Minecraft worlds. 
Minecraft is a very popular voxel-based game with procedural world generation. Minecraft's world format is well-documented and easy to parse with
the help of some open source libraries. 

#### World Format

The world is composed of `.mca` files on disk.
In the `.xml` format `Chunk` tag, a list of region file paths may be provided alongside the usual `Block`'s:

```xml
<Objects>
    <Chunk>
        <Region>big/r.-1.-1.mca</Region>
        <Region>big/r.-1.1.mca</Region>
        <Region>big/r.-1.3.mca</Region>
    </Chunk>
</Objects>
```

- `.mca` files store one region.
- Regions store an area of `32x32` chunks.
- Chunks store `256` chunk sections, some of which may be unused/null. These are stacked vertically.
- Chunk sections store `16x16x16` voxels.

Simple formulas are used to get a voxel's position based on the coordinates of its region, chunk and chunks section.
The code below loads these files to memory (simplified to highlight important parts):

```cpp
FILE* fp = fopen(region_file_path.string().c_str(), "rb");
enkiRegionFile region_file = enkiRegionFileLoad(fp);
for (int i = 0; i < 1024; i++) { // we iterate each chunk in the region
    enkiChunkBlockData chunk = enkiNBTReadChunk(&stream);
    for (int j = 0; j < 256; j++) { // for each chunk section
        if (chunk.sections[j] == nullptr)
            continue;

        const auto palette = chunk.palette[j]; // for getting block type

        for (int x = 0; x < 16; x++) {
            for (int y = 0; y < 16; y++) {
                for (int z = 0; z < 16; z++) {
                    let block = Block(block_pos, 0, scene_mat_index); // convert to internal block format
                    terrain.blocks.emplace_back(block);
                }
            }
        }
    }
    enkiNBTFreeAllocations(&stream);
}
enkiRegionFileFreeAllocations(&region_file);
fclose(fp);
```

Each chunk also has a palette. This basically defines which voxels are mapped to which materials. This will later be useful in the materials section.

#### Materials & Textures

To get the voxels to look as they do in-game, we needed to extract the original texture data from the game. This is stored as JSON metadata and PNG images.
Since these are very game specific, I will choose to not go into much detail. Briefly, the metadata is keyed by tags like `minecraft:stone`, `minecraft:grass_block`. These tags can be extracted from the aforementioned chunk palette. For each metadata key, a standard diffuse material is generated.
The JSON file corresponding to the tag is used to read the correct `.png` files. Because **different sides can have different textures**, multiple images may
be assigned to the same material. 

Since the voxels don't have a true UV definition, a technique similar to triplanar rendering is used.

Using the voxel normal from the hit point data, we can determine which side of the voxel is hit:

```cpp
TextureDirection dir;
if (images.contains(TextureDirection::All)) { // exception for voxels that have all sides the same
    dir = TextureDirection::All;
}
else if (normal.y < 0.0f) {
    dir = TextureDirection::Bottom;
}
else if (normal.y > 0.0f) {
    dir = TextureDirection::Top;
}
else {
    dir = TextureDirection::Side;
}
const auto image = images.at(dir).get(); // determine which image to sample
```

The obtained image is then sampled like usual.

## Results

Rendering a Minecraft world looks very beautiful. However, even though we rendered worlds from recent versions, the results actually looked pretty old-fashioned, something out of Infdev or Alpha era.

![Scenery](./images/scenery.jpg)

One of the nice things about our architecture is that we still have mesh support, so we tried to mix them to get this monstrosity:

![Scenery with dragon](./images/scenery_dragon.jpg)

This dragon looks a bit too bright, so here is the same scene with a better light/material combination:

![Scenery with better dragon](./images/scenery_shaded_dragon.jpg)

And lastly, and probably the most gorgeous one, a closeup image of the dragon:

![Closeup dragon](./images/closeup.jpg)

It is unfortunate that we have not been able to implement water correctly. One of the main reasons is that my (Erencan's) ray tracer did not anticipated coming to a dielectric material from another (or the same) one, so water blocks would refract rays again and again. Thus, we instead used full blue blocks, which looked so nostalgic.

## Performance

| Scene          | Init Time      | Render Time    |
|----------------|----------------|----------------|
| test.xml       | 0.50s          | 3.74s          |
| big_dragon.xml | 23.08s         | 32.98s         |
| closeup.xml    | 19.88s         | 41.18s         |

Initialization time scaled linearly with the number of regions, as expected. Because of the nature of Minecraft's world files, the loading process is a bit inefficient.

`closeup.xml` has 4 regions defined, amounting to a total of `129,010,458` blocks. This takes ~20 seconds to load. 
The efficiency of the SVO algorithm is demonstrated here, even with +100 million voxels the render time is still manageable.

We also ran profiling while rendering the `closeup.xml` scene:

![Call time graph](./images/graph.png)

The graph shows how much time each function took to execute at each recursion depth. 
The first huge hump is the SVO ray intersection function, the second is the SVO construction and the rest is the world loading.
A keen observer will notice the `log8` scaling of the SVO functions.

## Conclusion

What a term! 6 big homeworks have been a very daunting yet rewarding task for all of us, and now we even used it to create something about our passion of Minecraft! Actually, even though SVO idea came from me, Alp was more motivated than me due to the possibility of importing Minecraft worlds and rendering a Minecraft world using an HW6 ray/path tracer, and we really managed to do it. Optimizing the tree further, especially the traversal optimization by iterating the rays instead of intersecting with every block (which eliminates bounding box intersection and is similar to signed distance field ray marching, but the steps of node size instead of a distance function), better block support (like water or torches) and transformation and instancing support to chunks would be very good, yet we had some time constraints due to the HW6 unfortunately, causing the project to be less polished than we expected. However, this was a great experience, since sometimes we engineers have to create our work in such tight time intervals.

Speaking of engineering, these homeworks and projects have been pretty much the pinnacle of the knowledge we have gained in this degree program. From managing big codebases and maintaining to time constraints and sometimes-problematic-sometimes-not requirement specifications, this course has been pretty much **the** "Software Engineering" course. Thank you hocam for everything. This is a bit sad word to say since it has a meaning of not being able to meet again for some time, but farewell!

[^1]: Final reference, for the final homework! Since the ultimate goal of our project was rendering Minecraft worlds, this time I (Erencan) wanted to add a reference about Minecraft. The title of the post is a reference to one of the first live "sighting"s of Herobrine in Minecraft. Herobrine is a mythical being (so not in the actual game, or is it?) that exists inside Minecraft worlds. It is supposed to be something like a creepy stalker, watching the player from the fog, manipulating the parts of the world the player is currently not in (such as long 2x2 tunnels or removing leaves from trees) and disappearing whenever the player notices him. You can read a much more detailed history [here](https://minecraft.wiki/w/Herobrine#History). The reference is from the part of Patmuss' stream, where Copeland (the first person who "saw" him in a live stream) posted a [link](http://web.archive.org/web/20130710001351/http://ghostinthestream.net/him.html) (unfortunately, the original is lost, but here is the archived one). The link's name is "ghostinthestream.net", which is probably another reference to a classic anime [Ghost in the Shell](https://anilist.co/anime/43/GHOST-IN-THE-SHELL-Koukaku-Kidoutai/). The link has an animated GIF of Minecraft Steve's face, but the eyes are replaced with frantically rolling realistic ones, creating a pretty creepy look (I was actually scared from that GIF while reading the wiki page when I was a child). It also has a text mixed with question marks and non-Latin symbols. When you extract the Latin symbols, you get a modified version of another creepypasta:

[^2]: > It has been reported that some victims of torture, during the act, would retreat into a fantasy world from which they could not WAKE UP. In this catatonic state, the victim lived in a world just like their normal one, except they weren't being tortured. The only way that they realized they needed to WAKE UP was a note they found in their fantasy world. It would tell them about their condition, and tell them to WAKE UP. Even then, it would often take months until they were ready to discard  their fantasy world and PLEASE WAKE UP.

[^3]: This explanation actually became much longer than I anticipated, so let's finish it. If you want to learn more, the wiki page is a very good source.

---
{
    "author": ["Erencan Ceyhan", "Alp Beysir"],
    "lang": "en",
    "tags": ["C++", "Programlama", "Teknoloji", "Gönderi", "Ödev", "Işın İzleme"]
}
---

---
{
   "date": "[2025](/gönderiler/2025)-[01](/gönderiler/2025/01)-23 22:59:53+03:00"
}
---

