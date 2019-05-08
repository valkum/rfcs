# Table of Contents

- [Tracking Issues](#tracking-issue)
- [Amethyst Community Forum Discussion](#forum-discussion)
- [Motivation](#motivation)
- [Guide Level Explanation](#guide-level-explanation)
- [Reference Level Explanation](#reference-level-explanation)
- [Drawbacks]
- [Rationale and Alternatives](#rationale-and-alternatives)
- [Prior Art](#prior-art)
- [Unresolved Questions](#unresolved-questions)

# Basic Info
[basic]: #basic-info

- Feature Name: Terrain rendering (Vegaøyan)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty until a PR is opened)
- [Tracking Issue](#tracking-issue): (leave this empty)
- [Forum Thread](#forum-discussion): (if there is one)

# Summary
[summary]: #summary

Vegaøyan adds terrain rendering to Amethyst. Currently other big engines (Unreal Engine, Unity) provide their own terrain rendering 'components'. This RFC is partly inspired by [FarCry5's terrain rendering pipeline](https://www.gdcvault.com/play/1025480/Terrain-Rendering-in-Far-Cry) as well as a [report](https://victorbush.com/2015/01/tessellated-terrain/) by Victor Bush

Vegaøyan is a heightmap based terrain renderer utilizing a cardinal neighbor quadtree in combination with tessellation for LOD.
This RFC describes Amethyst's take on terrain rendering, introducing the amethyst_vegaøyan crate. The name Vegaøyan was picked from the Unesco World Heritage List and is a landscape in Norway.

This first version of Vegaøyan focuses on a CPU implementation. Further RFCs can take Vegaøyan to the GPU.

# Motivation
[motivation]: #motivation
Currently other big engines (Unreal Engine, Unity, Dunia Engine) provide their own terrain rendering 'components'. Amethyst currently lacks built-in support for this.

A built-in solution would:
- Help prototyping 3D games in Amethyst
- Help improving the adoption of Amethyst among indie developers
- Help getting Amethyst on feature parity with other big engines
- Act as a reference implementation for custom terrain rendering tied to a game
- Look good.

# Guide-Level Explanation
[guide-level-explanation]: #guide-level-explanation
A Terrain is a simple component which has material-like textures assigned to it. Basic Terrains use a heightmap, a normal map and a baked albedo map. Different settings are available to scale and offset the terrain (height-wise).

If added to an Entity the render pass uses the assigned textures to draw the terrain in your scene.
Based on the position of the active camera in the world, the level of detail for parts of the terrain is continuous updated to allow for large scale landscapes without overloading the GPUs processing power.

Open for discussion is if and how this information is passed to a physics system.




# Reference-Level Explanation
[reference-level-explanation]: #reference-level-explanation
<!-- <details>
<summary>The technical details and design of the RFC.</summary>
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
</details> -->
Large landscapes with adequate details require too many vertices to be covered by the built-in flat or pbr rendering pipelines. For this a additional pipeline is necessary. Terrain rendering can be split into different sub problems.
Level-of-Detail for different areas of the terrain, optimized assets for best performance and frustum culling to avoid creating draw commands for areas of the terrain that is e.g. behind the camera.

## LOD
The current design for Vegaøyan is based on a CPU based cardinal neighbor quadtree for LOD.

A quadtree is a 4-tree where a leaf is split into 4 new equal size leaves. Each leaf covers a specific region determined by an AABB.
For this design, splitting a leaf is done until one of two conditions is met. Either the configurable maximum depth is met or the distance between the camera and the center of a leaf is greater than `x` times the length of the edge of a leaf to its center.
A [cardinal neighbor quadtree](https://dx.doi.org/10.5120/ijca2015907501) is used instead of a normal quadtree to allow constant time lookup for neighbor leaves to get matching outer tessellation levels.

During the first update a System builds the cardinal neighbor quadtree based on the active camera's position. If the camera moves between two runs of the System more than `epsilon` units the quadtree is rebuild.

For each leaf a quad is rendered, translated and scaled according to the depth from the root node. Finally tessellation is applied to increase the detail of each quad and to get seamless edges between leaves of different depth values.

## Assets
The renderer needs 3 textures. A Heightmap, a normal map and an albedo map. A first step is to
<details>
<summary>Option A: The following information are from FarCry 5's terrain renderer.</summary>

__Heightmap__  
Format: R16_UNORM
Dimensions: 127x127
__Normal map__  
Format: BC3
Dimensions: 132x132
As we deal with terrain we can assume a positive z and thus can pack in smoothness and specular occlusion into the same texture.
__Albedo map__  
Format: BC1
Dimensions: 132x132
We can use 1-bit alpha channel for terrain cutouts, to support underground caverns.

Further RFCs should focus on asset streaming using Atelier.
</details>
<details>
<summary>Option B: Using megatextures</summary>
Having a megatexture for each texture.

__Heightmap__  
Format: R16_UNORM
Dimensions: Terrainsize  
__Normal map__  
Format: BC3
Dimensions: Terrainsize  
As we deal with terrain we can assume a positive z and thus can pack in smoothness and specular occlusion into the same texture.
__Albedo map__  
Format: BC1
Dimensions: Terrainsize
We can use 1-bit alpha channel for terrain cutouts, to support underground caverns.

</details>

## Culling
Culling can be done in 2D by checking if a circle/square around the center of a leaf is inside or outside of the 2D projection of the camera frustum.

# Drawbacks
[drawbacks]: #drawbacks

- We have to maintain an additional component
- Vegaøyan focuses only on 3D games with large enough terrains, where normal meshes are not sufficient

# Rationale and Alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- - Why is this design the best in the space of possible designs? -->
Different engines use different approaches to optimize terrain rendering. This approach is partly based on FarCry 5's terrain rendering which looks really good and offers good performance.
<!-- - What is the impact of not doing this? -->
If we don not add some kind of terrain rendering, each dev who needs some kind of large terrain has to implement a terrain renderer by themselves. This can be time consuming.
<!-- - What other designs have been considered and what is the rationale for not choosing them? -->
## Alternatives
- [Terrain Clipmap](https://developer.nvidia.com/gpugems/GPUGems2/gpugems2_chapter02.html)
Clipmap based terrain rendering uses more complex primitives to fill all gaps of the terrain. Furthermore it uses a coarse to fine interpolation thus fine details in terrain may be lost (??). Better suited for Simulators with real world height data.
I dropped work on this because of the more recent used techniques in AAA titles.
- [Unreal Engines Landscape](https://docs.unrealengine.com/en-us/Engine/Landscape/TechnicalGuide)
Unreal Engine uses multiple so called components per landscape. Each component has its own heightmap and can be further divided into 1-4 subsections. Each section results in a draw call. A section consists of up to 127 quads and is the base unit for LOD calculation. They use Mipmaps for LOD. Only components can be culled.
Our approach can cull single leaves (which would be sections in Landscape??) thus reduces the stress on the GPU better than Landscape.


# Prior Art
[prior-art]: #prior-art
<!-- <details>
<summary>Discuss previous attempts, both good and bad, and how they relate to this proposal.</summary>
A few examples of what this can include are:

- For engine, network, web, and rendering proposals: Does this feature exist in other engines and what experience has their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other engines, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other engines.
</details> -->

__Papers:__

[Slides on FarCry5's terrain rendering pipeline](https://www.gdcvault.com/play/1025480/Terrain-Rendering-in-Far-Cry)

__Comparable features in other engines:__

Pre-RFC: I have no experience with other engines terrain implementations. Maybe someone has and can provide some information in the RFC process?

# Unresolved Questions
[unresolved-questions]: #unresolved-questions
My current PoC uses a megatexture of 1024x1024px. FarCry uses small chunks for each quadtree leaf. We should collect pros/cons for each of the options.

My Cardinal Neighbor Quadtree implementation currently is arena based but really rough. The API is far from good. This needs obvious some work and discussion about API design.

The current design does not include decals, foliage etc. 

We should also consider working on support for tools. Until our editor is mature enough we can create some best practices for [Houdini](https://www.sidefx.com/products/houdini/) or [Gaea](https://quadspinner.com/Gaea)

Support for GPU only terrain rendering like in FarCry 5 should be addressed in the future.


Copyright 2018 Amethyst Developers