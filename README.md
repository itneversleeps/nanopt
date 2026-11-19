# nanopt

nanopt is a spectral path tracer written in C++20. Light transport is computed per-wavelength across the visible range (380–750 nm) instead of RGB. The "nano" is grounded: nanometers.

![Cornell box with a glass dragon, gold sphere, and a Veach-style row of equal-flux ceiling lights](references/m5-cornell.png)

## What's distinctive

Each path samples 5 wavelengths (one hero + four stratified secondaries) across 380–750 nm. Scene-time RGB literals are upsampled to spectra at evaluation time (Smits 1999); per-wavelength radiance integrates to CIE 1931 XYZ via Wyman 2013 analytic matching curves before mapping to D65-white linear sRGB on output. The payback for the spectral scope is dispersion: the glass dragon renders a visible rainbow that an RGB tracer cannot produce without faking it. The wavelength axis also makes iridescence and fluorescence tractable to add later, though neither is implemented yet.

The renderer has no dependency on Embree, OpenImageDenoise, or any third-party BSDF or sampling library — BVH traversal, importance sampling, and the OBJ loader are all implemented directly against the C++ standard library, with CMake as the only build dependency.

## Features

- **Geometry:** spheres, triangles, OBJ meshes with positions and per-vertex normals; flat or smooth shading via face-forward-clamped barycentric interpolation.
- **Acceleration:** SAH-built BVH; flat node array, ray-AABB slab traversal.
- **Materials**
  - *Lambertian:* RGB albedo, Smits-upsampled per evaluation.
  - *Rough conductor:* GGX D × height-correlated Smith G2 × per-wavelength complex Fresnel against measured η/κ tables (Au, Cu, Al; McPeak et al. 2015). Sampled via the GGX VNDF construction.
  - *Smooth dielectric:* perfect specular reflect/refract with wavelength-dependent IOR (Sellmeier 3-term; BK7 and SF10 ship). The refract branch is dispersive: it samples the hero direction only and terminates the secondary wavelengths for the rest of the path.
- **Lighting:** diffuse triangle area lights and delta point lights.
- **Direct lighting:** Veach two-strategy MIS: light-strategy NEE plus BSDF-strategy continuation, combined with the power-2 heuristic. Delta BSDFs route light through the BSDF strategy alone, so emitters refract through glass correctly.
- **Path tracer:** Russian-roulette-terminated multi-bounce GI, depth limit, throughput-based continuation probability.
- **Output:** PPM (P6, 8-bit, gamma-encoded).

## Build

Requires CMake 3.25+ and a C++20 compiler.

```sh
cmake -B build
cmake --build build --config Release
```

Run the resulting `nanopt` binary from the project root so it can find `assets/cornell-box.obj`. Output is written to `out.ppm` in the working directory.

## Reference renders

The Cornell box at every spectral milestone, kept as a visual regression baseline:

| | |
|---|---|
| ![M2 Cornell — RGB path tracing](references/m2-cornell.png) | ![M3 Cornell — through the spectral pipeline](references/m3-cornell.png) |
| **RGB path tracing baseline.** | **Same scene through the spectral pipeline** (5-wavelength sampling, Smits upsample, CIE → sRGB). Matches the RGB baseline to ±6/255 per channel on neutral materials. |
| ![M4 Cornell — Stanford XYZ RGB Dragon in dispersive SF10 glass](references/m4-cornell.png) | ![M5 Cornell — Veach-style row of equal-flux ceiling lights](references/m5-cornell.png) |
| **Stanford XYZ RGB Dragon in SF10 glass.** Visible chromatic dispersion through the body. | **Direct-lighting MIS demo.** Equal-flux ceiling lights at decreasing size; small lights favour light sampling, large lights favour BSDF sampling, MIS picks per pixel. |

## Roadmap

Tile-based parallel rendering, EXR output, and prose documentation of the underlying math — light transport equation, spectral pipeline, BVH layout, BRDF reference.

## Acknowledgements

- **Stanford XYZ RGB Dragon:** Stanford 3D Scanning Repository (http://graphics.stanford.edu/data/3Dscanrep/). Distributed via the [common-3d-test-models](https://github.com/alecjacobson/common-3d-test-models) collection.
- **Conductor optical constants:** K. M. McPeak, S. V. Jayanti, S. J. P. Kress, S. Meyer, S. Iotti, A. Rossinelli, D. J. Norris, *"Plasmonic films can easily be better: Rules and recipes,"* ACS Photonics 2:326-333 (2015). Tables resampled from the CC0 data hosted at [refractiveindex.info](https://refractiveindex.info).

## License

MIT — see [LICENSE](LICENSE).
