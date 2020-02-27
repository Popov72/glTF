# KHR\_materials\_sheen

## Khronos 3D Formats Working Group

* TODO

## Acknowledgments

* TODO

## Status

Experimental

## Dependencies

Written against the glTF 2.0 spec.

## Exclusions
* This extension must not be used on a material that also uses `KHR_materials_pbrSpecularGlossiness`.
* This extension must not be used on a material that also uses `KHR_materials_unlit`.

## Overview

This extension defines a sheen that can be layered on top of an existing glTF material definition. A sheen layer is a common technique used in Physically-Based Rendering to represent cloth and fabric materials, for example. See [Theory, Documentation and Implementations](#theory-documentation-and-implementations)

## Extending Materials

The PBR sheen materials are defined by adding the `KHR_materials_sheen` extension to any compatible glTF material (excluding those listed above). 
For example, the following defines a material like velvet.

```json
{
    "materials": [
        {
            "name": "velvet",
            "extensions": {
                "KHR_materials_sheen": {
                    "intensityFactor": 0.9
                }
            }
        }
    ]
}
```

### Sheen

All implementations should use the same calculations for the BRDF inputs. Implementations of the BRDF itself can vary based on device performance and resource constraints. See [appendix](/specification/2.0/README.md#appendix-b-brdf-implementation) for more details on the BRDF calculations.

|                                  | Type                                                                            | Description                            | Required                       |
|----------------------------------|---------------------------------------------------------------------------------|----------------------------------------|--------------------------------|
|**intensityFactor**               | `number`                                                                        | The sheen intensity.                   | No, default: `1.0`             |
|**colorFactor**                   | `array`                                                                         | The sheen color in linear space        | No, default: `[1.0, 1.0, 1.0]` |
|**colorIntensityTexture**         | [`textureInfo`](/specification/2.0/README.md#reference-textureInfo)             | The sheen color (RGB) and intensity (Alpha) texture.<br> The sheen color is in sRGB transfer function | No               |
|**roughnessFactor**               | `number`                                                                        | The sheen roughness.                   | No, default: `0.0`             |

The sheen roughness is independent from the material roughness to allow materials like this one, with high material roughness and small sheen roughness:

![Cushion](./figures/cushion.png)

If a texture is defined: 
* The sheen intensity is computed with : `sheenIntensity = intensityFactor * sample(colorIntensityTexture).a`.
* The sheen color is computed with : `sheenColor = colorFactor * sampleLinear(colorIntensityTexture).rgb`.

Otherwise, `sheenIntensity = intensityFactor` and `sheenColor = colorFactor`

The sheen formula `f_sheen` is a new BRDF, different from the specular and clear coat BRDF:
```glsl
sheenTerm = sheenColor * sheenIntensity * sheenDistribution * sheenVisibility;
```

As you notice, there is no fresnel term for the sheen layer.

### Sheen distribution

The sheen distribution follows the "Charlie" sheen definition from ImageWorks (Estevez and Kulla):
```glsl
alphaG = sheenRoughness * sheenRoughness
invR = 1 / alphaG
cos2h = NdotH * NdotH
sin2h = 1 - cos2h
sheenDistribution = (2 + invR) * pow(sin2h, invR * 0.5) / (2 * PI);
```

### Sheen visibility

The "Charlie" sheen visibility is also defined in the same document:
```glsl
float l(float x, float alphaG)
{
    float oneMinusAlphaSq = (1.0 - alphaG) * (1.0 - alphaG);
    float a = mix(21.5473, 25.3245, oneMinusAlphaSq);
    float b = mix(3.82987, 3.32435, oneMinusAlphaSq);
    float c = mix(0.19823, 0.16801, oneMinusAlphaSq);
    float d = mix(-1.97760, -1.27393, oneMinusAlphaSq);
    float e = mix(-4.32054, -4.85967, oneMinusAlphaSq);
    return a / (1.0 + b * pow(x, c)) + d * x + e;
}

float lambdaSheen(float cosTheta, float alphaG)
{
    return abs(cosTheta) < 0.5 ? exp(l(cosTheta, alphaG)) : exp(2.0 * l(0.5, alphaG) - l(1.0 - cosTheta, alphaG));
}

sheenVisibility = 1.0 / ((1.0 + lambdaSheen(NdotV, alphaG) + lambdaSheen(NdotL, alphaG)) * (4.0 * NdotV * NdotL));
```

However, depending on device performance and resource constraints, one can use a simpler visibility term, like the one defined by Ashikhmin (but that will make the BRDF not energy conserving when using the albedo-scaling technique described below):
```glsl
sheenVisibility = 1 / (4 * (NdotL + NdotV - NdotL * NdotV))
```

### Sheen layering

#### Albedo-scaling technique

The sheen layer can be combined with the base layer with an albedo-scaling technique described in Estevez and Kulla:

<blockquote>
float max3(vec3 v) { return max(max(v.x, v.y), v.z); }

albedoScaling = min(1.0 - sheenIntensity * max3(sheenColor) * E(VdotN), 1.0 - sheenIntensity * max3(sheenColor) * E(LdotN))

f = f<sub>sheen</sub> + f<sub>base</sub> * albedoScaling
</blockquote>

The values `E(x)` can be looked up in a table which can be found in section 6.2.3 of [Enterprise PBR Shading Model](#theory-documentation-and-implementations) if you use the "Charlie" visibility term. If you use Ashikhmin instead, you can get the lookup table by using the [cmgen tool from Filament](#theory-documentation-and-implementations), with the `--ibl-dfg` and `--ibl-dfg-cloth` flags: the table is in the blue channel of the generated picture. The lookup must be done with `x = VdotN` and `y = sheenRoughness`.

If you want to trade a bit of accuracy for more performance, you can use the `VdotN` term only and thus avoid doing multiple lookups for `LdotN`. The albedo scaling term is simplified to:
<blockquote>
albedoScaling = 1.0 - sheenIntensity * max3(sheenColor) * E(VdotN)
</blockquote>

In this simplified form, it can be used for both direct and indirect lights:
```glsl
specular_direct *= albedoScaling;
diffuse_direct *= albedoScaling;
environmentIrradiance_indirect *= albedoScaling
specularEnvironmentReflectance_indirect *= albedoScaling
```

#### Simple layering technique

However, due to performance constraints, one can use a simpler layering technique (which is energy conserving). For example:
```glsl
specularity = mix(metallicF0, baseColor, metallic)
reflectance = max(max(specularity.r, specularity.g), specularity.b)

final = f_emissive + f_diffuse + f_specular + (1 - reflectance) * f_sheen
```
Note that this type of layering links the strength of the effect to the color of the base material, which may be confusing to the user.

`f_sheen` is the sheen contribution computed from all light sources in the scene (sum of all `sheenTerm * lightColor * NdotL` terms for direct lights and `sheenIntensity * sheenEnvironmentRadiance * sheenEnvironmentReflectance` for indirect light).

If `intensityFactor = 0`, the sheen layer is disabled and the material is behaving like the core material:

```glsl
f = f_emissive + f_diffuse + f_specular
```
  
## Reference

### Theory, Documentation and Implementations

[Filament Material models - Cloth model](https://google.github.io/filament/Materials.md.html#materialmodels/clothmodel)
[cmgen tool from Filament](https://github.com/google/filament)
[NP13 David Neubelt, Matt Pettineo – “Crafting a Next-Gen Material Pipeline for The Order: 1886”, SIGGRAPH 2013](https://blog.selfshadow.com/publications/s2013-shading-course/rad/s2013_pbs_rad_notes.pdf)
[EK17 Alejandro Conty Estevez, Christopher Kulla – “Production Friendly Microfacet Sheen BRDF”, SIGGRAPH 2017](https://blog.selfshadow.com/publications/s2017-shading-course/imageworks/s2017_pbs_imageworks_sheen.pdf)
[AS07 Michael Ashikhmin, Simon Premoze – “Distribution-based BRDFs”, 2007](http://www.cs.utah.edu/~premoze/dbrdf/dBRDF.pdf)
[cloth-shading](https://knarkowicz.wordpress.com/2018/01/04/cloth-shading/)
[Enterprise PBR Shading Model - Sheen](https://dassaultsystemes-technology.github.io/EnterprisePBRShadingModel/spec-2021x.md.html#components/sheen)