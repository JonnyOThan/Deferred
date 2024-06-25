# Deferred - Adding deferred rendering to KSP

You'll find an [explanation of what the mod does here](https://www.patreon.com/posts/deferred-106557481).

# Dependencies

Shabby is needed (currently bundled on the github release but will be separate in future releases and CKAN).

# Limitations/known issues
## Terrain shaders
Only "Ultra" quality terrain shaders are compatible, that includes both atlas and non-atlas terrain shaders. Terrain quality is forced to Ultra when the mod loads.

Names of compatible terrain shaders:

 - PQSTriplanarZoomRotation.shader
 - PQSTriplanarZoomRotationTextureArray - 1Blend.shader
 - PQSTriplanarZoomRotationTextureArray - 2Blend.shader
 - PQSTriplanarZoomRotationTextureArray - 3Blend.shader
 - PQSTriplanarZoomRotationTextureArray - 4Blend.shader

## Transparency
Traditional transparency doesn't work in deferred rendering for opaque objects (only used in the editors in KSP). To emulate transparency, a stylized dissolve effect is used on fairing-only shaders, and a dithering effect is applied on regular shaders.

![enter image description here](https://i.imgur.com/RIjNtSZ.png)

# Mod compatibility status
In no particular order.
Mods that say "renders in forward" means they may appear to render correctly but get no deferred benefits for now (no lighting perf improvements, not compatible with any deferred ambient/lighting/effects)

| Mod  | Status | Notes |
| ------------- | ------------- |------------- |
| Textures Unlimited  |	Compatible [via fork](https://github.com/LGhassen/TexturesUnlimited/releases), otherwise renders in forward |	|
| Parallax  | Incompatible, mix of artifacts and rendering in forward, will be compatible soon |
| Conformal decals  | Renders in forward, will be fully compatible soon |
| Scatterer | Compatible |
| EVE-Redux | Compatible |
| Volumetric clouds (and related Scatterer versions) | Fixed individual DLLs [can be downloaded here for v3 and v4](https://drive.google.com/drive/folders/1lkJWJ6qfWLdJt2ZYjTYuOQk3dO7zxMCb?usp=sharing), or full updated downloads are provided on Patreon if you still have access. v1 and v2 appear to be compatible |
| TUFX | Compatible
| Shaddy | Renders in forward
| Kopernicus | Untested, same limitations to terrain shaders apply as stock (only stock Ultra terrain shaders supported for now)
| Waterfall | Compatible
| FreeIVA | Compatible
| KerbalVR | Compatible
| PlanetShine | Compatible but obsolete at default settings. Use if you want more control over lighting, have custom settings and know what you are doing
| SimpleAdjustableFairings  | Compatible
| KerbalKonstructs | Compatible
| RasterPropMonitor | Compatible
| Engine Lighting | Compatible
| B9 Procedural Wings | Renders in forward
| KSRSS  | Incompatible, black terrain
| NeptuneCamera  | Incompatible
| Camera mods | Unknown/untested

# Debug menu
Using alt + d ( right shift + d on linux) will bring up a simple debug menu cycling between the contents of the g-buffer (albedo, normals, smoothness, specularColor, occlusion) and a composite of the emission+calculated ambient

Transparencies and incompatible forward shaders will render on top of the debug visualization, ignoring the g-buffer mode selected. This can also be used to identify incompatible/forward shaders (ignoring transparencies)

![enter image description here](https://i.imgur.com/ZgSDZnu.jpeg)

# Stencil buffer usage
Only 4 bits of the stencil buffer appear to be available in deferred rendering because reasons.
Stencil buffer is useful for applying post effects selectively to certain surfaces, we can take advantage of it here since we are using new shaders and can implement stencil everywhere. I propose the following stencil values be used for masking, they are already used by this mod for replaced shaders:

| Surface/Shader type | Stencil bit| Stencil value | Notes |
| ------------- | ------------- |------------- |------------- |
| Terrain (stock/parallax)  | 0	| 1 | Already used in this mod to emulate the alpha PQS to scaled fade, since it is impossible to do alpha blending otherwise in deferred (dithering looked really bad here and caused other issues with visual mods)|
| Parallax grass | 1 |	2 | Parallax grass has normals that point upwards, matching the terrain and not the grass itself so it might be worthwhile to have a separate stencil value for it, for any image effects that might need accurate normals|
| Local scenery (buildings + stock/parallax scatters)  | 2|	4 | |
| Parts  | 3|	8 | |
## Writing stencil values

To write stencil values from a shader, add a stencil block with the stencil value to write, e.g for parts:

        Tags { "RenderType"="Opaque" }

        Stencil
        {
            Ref 8
            Comp Always
            Pass Replace
        }  

        CGPROGRAM
        ...

## Testing stencil values

For testing/checking stencil values in a post effect, combine the stencil bits you want to test for, use that as ReadMask and verify that the result is superior to zero, using Comp and Ref as seen in https://docs.unity3d.com/Manual/SL-Stencil.html
Examples:
To check for Parts:

            Stencil
            {
                ReadMask 8
                Comp Greater
                Ref 0
                Pass Keep
            }

To check for Scenery and terrain, combine bits 2 and 0:

            Stencil
            {
                ReadMask 5
                Comp Greater
                Ref 0
                Pass Keep
            }

To check for terrain and only terrain

            Stencil
            {
                ReadMask 1
                Comp Equal
                Ref 1
                Pass Keep
            }



