# CENG795-HW5: High Dynamic Raytracing

Hello again minasan! Welcome again to yet another CENG795 homework post. Unfortunately, this post will not be that nice compared to the previous ones. HW3, HW4 and HW5 have been all pretty easy to do. In this one however, unlike the others, I have not done any "adventurous" thing in this one unfortunately; nevertheless, I will try to do my best to make this post interesting.

## Implementation

### Material Refactor

I have started doing some refactoring since the previous homework, as I had written in the blog post of the previous homework. In this homework, I have pretty much completed the refactoring by migrating material system from `std::variant` to virtual methods. In the [performance](#performance) section, you can see a comparison between the two implementations, a benchmark I did before implementing HDR support.

The new architecture is pretty similar to the texture architecture I had implemented in the previous homework. There is an interface (or an abstract class, since C++ does not have interfaces) named `IMaterial`. It is defined as this:

```cpp
struct IMaterial {
    virtual glm::vec3 shadeMaterial(const shading_arguments& args) const = 0;
    glm::vec3 shade(const shading_arguments& args) const;
};
```

The main work is done by `shadeMaterial`, which is overriden by the child classes. The `shade` method (or _mixin_) just does recursion depth check and calls `shadeMaterial` if it is not exceeded. I made it a separate method since otherwise, this check would be the first line in every override of `shadeMaterial`.

The data is defined in semi-POD `struct`s (semi since non-default constructors break this). The `struct` of simple materials is as follows:

```cpp
struct DSimple {
    glm::vec3 ambient_reflectance = {};
    glm::vec3 diffuse_reflectance = {};
    glm::vec3 specular_reflectance = {};
    float roughness = 0;
    float phong_exponent = 0;

    DSimple() = default;
    DSimple(
        glm::vec3 ambient_reflectance,
        glm::vec3 diffuse_reflectance,
        glm::vec3 specular_reflectance,
        float roughness,
        float phong_exponent
    );
    glm::vec3 simpleShade(const shading_arguments& args) const;
    glm::vec3 reflectShade(const shading_arguments& args) const;
};
```

The data is just what each material has, there is a constructor taking each of these so that I can construct an instance directly. I also re-defined default constructor since defining another constructor removes the default one. I also created some helper methods for shading. The other materials are pretty much the same except there is no helper methods and they inherit from `DSimple`. The real `struct`s are defined similar to `Simple`, which is defined as follows:

```cpp
struct Simple : DSimple, IMaterial {
    Simple() = default;
    Simple(const DSimple& simple);
    glm::vec3 shadeMaterial(const shading_arguments& args) const override;
};
```

It takes the data as itself and copies it. The `IMaterial` is there to provide the `shade` and polymorphism.

The scene now holds `std::unique_ptr`s if `IMaterial`s, which actually decreases cache efficiency since now the materials are not actually contiguous; however, this should not decrease the performance that much, since materials are not used consecutively anyways, they are used based on which object a ray hits. There is usually not much of a spatial locality in material access, but there is temporal locality (since neighboring rays have more chance to hit an object of same material), and storing pointers does not decrease it more than using sum types. This actually did not change the number of lines that much, but now the code is much cleaner and modularized, which is a huge bonus.

### Directional Lights

The next thing I did was doing directional lights. However, in my generic `diffuse` and `specular` functions, which calculate the diffuse and specular shading results, respectively, I took the position of the light, which is the real position of the point light and the sampled point for area lights. The shadow ray is also calculated using the position of the light. However, directional lights _do not_ have positions! What I did was pretty much a hack. Shadow casts check for whether the intersection distance (which equals `t` since direction is a unit vector) is smaller than the distance between the light and the origin of the ray. If there is no hit, the point is not in shadow. If there is a hit but the hit is farther than the light, there is still no shadow. The code was this:

```cpp
static bool castShadowRay(const shading_arguments& args, glm::vec3 target) {
    auto hi = HitInfo();
    const auto origin = args.hit_point + args.n * args.s.shadow_ray_epsilon;
    const auto sent_ray = Ray{.origin = origin, .direction = glm::normalize(target - origin), .time = args.ray_time};
    return (not args.s.rayCast<false>(sent_ray, hi) or hi.t > glm::length(target - origin));
}
```

It creates a `HitInfo` `struct`, the ray and casts it in the scene without backface culling (denoted by `<false>`). The logic is the same as I said in the previous paragraph. Solving this was actually a bit harder than solving the shading. In the shading, I created a new point, which is in the opposite direction of the direction of the light from the hit point. This is something like the origin point of the light, but the distance between the hit point and light position is always parallel to the light's direction, which is the whole point (pun not intended) of the directional light.

```cpp
const auto virtual_light_position = args.hit_point - light.direction;
if (not castShadowRay(args, virtual_light_position)) {
    continue;
}
const auto diffuse_value =
    diffuse(virtual_light_position, light.radiance, diffuse_reflectance, args.n, args.hit_point);
const auto specular_value = specular(
    virtual_light_position,
    light.radiance,
    specular_reflectance,
    args.n,
    args.hit_point,
    args.camera_pos,
    phong_exponent
);
ret += diffuse_value + specular_value;
```

The `virtual_light_position` is the point I have talked about in the previous paragraph. This piece of code is inside a `for` loop over directional lights, so the `continue` just means a skip of this light due to shadow.

However, the result is this:

![Cube under Cropped Directional Light](./assets/cube_directional_distance_check.png)

As you can see, the shadow is cut at the right of the image. After some debugging, I found out that this was due to the fact that the `castShadowRay` function thinks that there is no shadow at those places since the ray hit (which is on the cube) is not in front of the light, which is at position `virtual_light_position`, so the distance is `1`. Solving this was a bit harder than solving the shading, but still pretty easy. After changing the `castShadowRay` to this:

```cpp
static bool castShadowRay(const shading_arguments& args, glm::vec3 target, bool check_distance = true) {
    auto hi = HitInfo();
    const auto origin = args.hit_point + args.n * args.s.shadow_ray_epsilon;
    const auto sent_ray = Ray{.origin = origin, .direction = glm::normalize(target - origin), .time = args.ray_time};
    return (not args.s.rayCast<false>(sent_ray, hi) or (check_distance and hi.t > glm::length(target - origin)));
}
```

and calling it with `false` as the third argument _only in directional lights_ solves this issue.

![Cube under Directional Light](./assets/cube_directional.png)

Yep, the directional light was this!

### Spot Light

Spot lights are a bit harder than directional lights since they need a bit of maths. Since the angle between two vectors can easily be computed using dot products if both of vectors are normalized, which can be the case since the direction of the spotlight can be normalized in parsing, which I did. However, the result was this:

![Wrong Cube under Spot Light](./assets/cube_point_wrong.png)

![Wrong Dragon under Spot Light](./assets/dragon_spot_light_msaa_light_falloff_instead_of_coverage_cos_half.png)

There was definitely a calculation error here. After some painful debugging sessions, I found out the culprit:

```cpp
(cos_a - light.falloff_cos_half) / (light.falloff_cos_half - light.coverage_cos_half)
```

or in math notation

$$
\frac{\cos(\alpha) - \cos(\text{falloff} / 2)}{\cos(\text{falloff}/2) - \cos(\text{coverage}/2)}
$$

This is what is written in the homework PDF and it has a small error: the substraction in the nominator should be between $\cos(\alpha)$ and $\cos(\text{coverage}/2)$, not $\cos(\text{falloff}/2)$. After fixing that error, I got the correct result:

![Dragon under Spot Light](./assets/dragon_spot_light_msaa.png)

Unfortunately, I don't have the ground truth for cube under spotlight since it is a case I created to test spotlight. It shows the errors, but I cannot know whether it is rendered correctly. However, I do not think there should be an issue since dragon is rendered correctly, almost pixel perfect.

### Tonemapping

Abstractions, abstractions, abstractions. So far, the tonemapping (which was simple clamping) had been done in the last step right before writing the color to the image. However, tonemapping needs the HDR values, not 8-bit values, which the image was. Hence, I did an abstraction and created an interface, `ITonemapper`. The `DClamp` (and `Clamp`) and `DReinhard` (and `Reinhard`) are defined similar to my previous classes. `Clamp` implements the usual clamping while `Reinhard` implements the Photographic TMO. `Clamp` currently takes saturation and gamma; however, it does not use this since `Clamp` simulates previous homeworks, which did not have gamma and saturation. `Reinhard` on the other hand handles both of them.

Implementing the TMO was also pretty easy, I pretty much implemented the algorithm from the slides and the global operator is pretty easy. However, the result was a bit off.

![Too Bright Sphere with HDR Texture and Artifacts](./assets/sphere_point_hdr_texture_delta.png)

The image was somewhat brighter than the instructor's output. Furthermore, the lights had weird artifacts, their brightest parts were colored instead of being white. After long (longest in this homework) and painful debugging and toilet sessions, the God revealed me the reason, which was clamping the luminance. Such a simple mistake, and the output was wrong in such an obvious way!

While tonemapping each pixel, I had this code in the `for` loop:

```cpp
const auto luminance = glm::dot(image[i], luminancer);
const auto L = zone_factor * luminance;
const auto L_d = glm::clamp(float(L / (1.0 + L) * (1.0f + L / L_w_2)), 0.0f, 1.0f);
const auto value =
    glm::clamp(L_d * glm::pow(image[i] / luminance, glm::vec3(saturation)), glm::vec3(0.0f), glm::vec3(1.0f));
auto gamma_corrected = glm::pow(value, glm::vec3(degamma));
ret[i] = glm::u8vec3(255.0f * glm::clamp(gamma_corrected, 0.0f, 1.0f));
```

The `luminancer` is defined as this:

```cpp
constexpr auto luminancer = glm::vec3(0.2126f, 0.7152f, 0.0722f);
```

which is the weights of each channel in luminance calculation. `zone_factor` is the `key`, one of the parameters of the TMO, divided by the geometric average of the luminances. This result would then be multiplied to get the `L` in the formula, which is then mapped using the global TMO with burn-out. However, as one can see, the resulting `L_d` is clamped to `[0,1]` range. I did not see this as an error, but when I removed it, those artifacts were gone!

![Too Bright Sphere with HDR Texture](./assets/sphere_point_hdr_texture_only_delta.png)

However, the image was still brighter than the ground truth. After a bit of experimenting, I found out that the $\delta$ in the geometric average changed the result considerably, brightening or darkening the whole image. I guess the reason is that the image has too many black pixels (with luminance `0`), which means that the $\delta$ was significant, since it affected the result. After binary searching, I found out that my $\delta$ value of $2.5\times 10^{-4}$ more-or-less corresponds to the ground truth. The result was almost pixel perfect copy of the ground truth image:

![Correct Sphere with HDR Texture](./assets/sphere_point_hdr_texture_correct.png)

I also used tinyEXR to handle EXR files, since the HDR textures were given as EXR files and stb does not support EXR files, only HDR files. Currently, the scene is always exported as both PNG (tonemapped with clamping, or Reinhard if it is defined) and HDR (not EXR). This means that one of the other small but significant parts of the homework. My current raytracer however _always_ exports HDR, not when the output is EXR or HDR, it also does not export EXR. This is pretty easy to implement but network homework, term project and New Year decreased my time budget for this homework.

### HDR Environment Map

And here comes the last part of the homework! Even though improving the outputs' visuals massively, implementing the environment maps was also straightforward like the previous parts of the homework. Until now, there have been no code in the `lights.cpp`, since lights had been handled in the shading function, which is probably a bit bad in terms of architecturing the program but it have worked. However, since sampling an environment map is also done to sample the background, I decided to make it its own function. It is defined as the follows:

```cpp
glm::vec3 rt::EnvironmentLight::sample(glm::vec3 direction) const {
    const auto d = direction;
    auto uv = glm::vec2();
    switch (sampling) {
        case rt::EnvironmentLight::Sampling::LatLong: {
            uv[0] = (1.0f + std::atan2(d.x, -d.z) / std::numbers::pi) / 2.0f;
            uv[1] = std::acos(d.y) / std::numbers::pi;
        } break;
        case rt::EnvironmentLight::Sampling::Probe: {
            const auto r = std::acos(-d.z) / (std::numbers::pi * std::sqrt(d.x * d.x + d.y * d.y));
            uv[0] = (r * d.x + 1.0f) / 2.0f;
            uv[1] = (-r * d.y + 1.0f) / 2.0f;
        } break;
    }
    uv *= glm::vec2(image.width, image.height);
    return image.sample(uv.x, uv.y);
}
```

It takes a direction vector and converts it to UV coordinates using the proper formula and samples the texture. Again, this was a piece of code that worked at the first time, giving me the rare programmer satisfaction yet again.

However, I did not use rejection sampling at first, since the non-uniform radial sampling was much easier to implement, while giving similar results. I created a `getRandomDirection` function in order to modularize it so that I could change it later, which would be really handy. The code was this:

```cpp
static glm::vec3 getRandomDirection() {
    const auto theta = getRandomNumber(0.0f, 2.0f * std::numbers::pi);
    const auto phi = getRandomNumber(0.0f, std::numbers::pi);
    return glm::vec3(std::sin(phi) * std::cos(theta), std::sin(phi) * std::sin(theta), std::cos(phi));
}
```

where `getRandomNumber` gives a random uniform number in the given range. I changed this function to a proper rejection sampling one _after_ finishing the other parts of the homework. The final `getRandomDirection` is this:

```cpp
static glm::vec3 getRandomDirection() {
    auto dir = glm::vec3(getRandomNumber(-1.0f, 1.0f), getRandomNumber(-1.0f, 1.0f), getRandomNumber(-1.0f, 1.0f));
    while (glm::dot(dir, dir) > 1.0f) {
        dir = glm::vec3(getRandomNumber(-1.0f, 1.0f), getRandomNumber(-1.0f, 1.0f), getRandomNumber(-1.0f, 1.0f));
    }
    return dir;
}
```

I also measured the performance of both. The result was that random rejection sampling is noticeably slower but not that much. The results are in the [performance](#performance) section.

At first, I forgot to use the sampled value as a light source and instead directly set the final value, which resulted in this:

![Sphere under Environment Light without Shading](./assets/sphere_env_light_no_shading.png)

I immediately realized this mistake and fixed it easily by considering the result as a directional light coming from the sampled direction. However, there was still a problem since the output looked like a black hole:

![Black Hole under Environment Light](./assets/sphere_env_light_black_hole.png)

![Another Black Hole under Environment Light (This Time Starry Space)](./assets/black_hole.png)

After looking at the black hole and seeing my future in its singularity, I found out that I took the direction of the light ray in the opposite direction. In the directional light, the direction is the light's direction, so I substracted the direction from the hit point. However, in environment map, the direction is _towards_ the light, so I need to add them instead of substracting. After fixing this, the result looked pretty nice:

![Mostly Correct Sphere under Environment Light](./assets/sphere_env_light_biased_sampling.png)

The next case I tried was the head. It looks soooooooooo nice! I also tried it with different sampling rates: 1, 4, 9, 25, 64, 81, 256, 400, 625, 900. I also created an animated GIF showing the improvement with increasing sampling rate.

![Improving Head under Environment Light](./assets/head_env_light_biased.gif)

The result is a bit off, and it also do not use smooth shading, but the progression looks very nice.

I also tried the end of homework boss: Audi.

![Wrong Audi under Glacier Environment Light](./assets/audi-tt-glacier_texture_error.png)

Do you see the problem? The car has a 『　　』[^1] plate! After some debugging, I found out that some texture coordinates were negative, which broke the program. I used wrapping instead of clamping in texture coordinates and the result became mostly correct.

![Mostly Correct Audi under Glacier Environment Light](./assets/audi-tt-glacier_inverse_map_instead_of_rejection.png)

So, the homework is done! I fixed up the sampling, whose results are in the [results](#results).

## Results

![Correct Audio under Glacier Environment Light except Noise](./assets/audi-tt-glacier.png)

![Correct Audio under Pisa Environment Light except Noise](./assets/audi-tt-pisa.png)

![Empty Environment with LatLong Texture](./assets/empty_environment_latlong.png)

![Empty Environment with Probe Texture](./assets/empty_environment_light_probe.png)

![Spherical Glass under Environment Light](./assets/glass_sphere_env.png)

![Spherical Mirror under Environment Light](./assets/mirror_sphere_env.png)

## Performance

First, the benchmark between virtual and variant. Since `std::variant`'s implementation can dramatically change between the compilers, I also tested between compilers.

| Testcase                             | Rendering Time (GCC, virtual) | Rendering Time (MSVC, virtual) | Rendering Time (GCC, variant) | Rendering Time (MSVC, variant) |
| ------------------------------------ | ----------------------------- | ------------------------------ | ----------------------------- | ------------------------------ |
| bunny.xml                            | 0.0283802                     | 0.0458557                      | 0.0287202                     | 0.0468452                      |
| cornellbox_recursive.xml             | 0.229955                      | 0.36028                        | 0.238443                      | 0.371436                       |
| other_dragon.xml                     | 0.361675                      | 0.526551                       | 0.361571                      | 0.525055                       |
| scienceTree_glass.xml                | 0.463028                      | 0.852566                       | 0.478199                      | 0.870111                       |
| simple.xml                           | 0.0219692                     | 0.0374878                      | 0.0220329                     | 0.0400985                      |
| two_spheres.xml                      | 0.0210437                     | 0.0283466                      | 0.0223661                     | 0.0281506                      |
| instance/ellipsoids.xml              | 0.0788359                     | 0.119194                       | 0.081917                      | 0.122259                       |
| instance/metal_glass_plates.xml      | 0.664058                      | 1.11338                        | 0.677238                      | 1.15823                        |
| instance/spheres.xml                 | 0.0556113                     | 0.0909408                      | 0.0594592                     | 0.094961                       |
| dist/cornellbox_area.xml             | 8.36556                       | 14.7246                        | 9.2073                        | 15.0258                        |
| dist/focusing_dragons.xml            | 38.455                        | 58.5525                        | 33.8644                       | 58.6734                        |
| dist/spheres_dof.xml                 | 5.44902                       | 8.11655                        | 5.78713                       | 8.19425                        |
| texture/bump_mapping_transformed.xml | 0.121709                      | 0.139171                       | 0.131904                      | 0.148202                       |
| texture/brickwall_with_normalmap.xml | 0.0846417                     | 0.16104                        | 0.0862123                     | 0.165274                       |
| texture/cube_cushion.xml             | 0.083873                      | 0.157696                       | 0.0838197                     | 0.16078                        |
| texture/cube_perlin.xml              | 0.12215                       | 0.275763                       | 0.124521                      | 0.272554                       |
| texture/cube_perlin_bump.xml         | 0.138111                      | 0.292683                       | 0.137821                      | 0.294797                       |
| texture/cube_wall.xml                | 0.0778775                     | 0.152509                       | 0.0790112                     | 0.156742                       |
| texture/cube_wall_normal.xml         | 0.0841438                     | 0.159097                       | 0.0866704                     | 0.163564                       |
| texture/cube_waves.xml               | 0.0809337                     | 0.157239                       | 0.0843926                     | 0.162292                       |
| texture/ellipsoids_texture.xml       | 0.0771365                     | 0.117501                       | 0.0793805                     | 0.118619                       |
| texture/galactica_static.xml         | 0.367829                      | 0.72849                        | 0.376033                      | 0.738424                       |
| texture/killeroo_bump_walls.xml      | 0.26567                       | 0.459497                       | 0.270957                      | 0.466517                       |
| texture/plane_bilinear.xml           | 0.0220511                     | 0.0301912                      | 0.0278078                     | 0.0332584                      |
| texture/sphere_nearest_bilinear.xml  | 0.0813739                     | 0.0842668                      | 0.0894229                     | 0.0906545                      |
| texture/sphere_nobump_bump.xml       | 0.0874026                     | 0.0945517                      | 0.0956179                     | 0.101027                       |
| texture/sphere_nobump_justbump.xml   | 0.0889195                     | 0.0929987                      | 0.0929962                     | 0.0978418                      |
| texture/sphere_perlin.xml            | 0.126511                      | 0.135037                       | 0.14301                       | 0.144179                       |
| texture/sphere_perlin_bump.xml       | 0.155357                      | 0.163569                       | 0.160708                      | 0.165377                       |
| texture/sphere_perlin_scale.xml      | 0.129742                      | 0.136017                       | 0.143759                      | 0.145046                       |
| texture/wood_box.xml                 | 0.0785099                     | 0.153957                       | 0.0816696                     | 0.158564                       |
| texture/wood_box_all.xml             | 0.0929561                     | 0.170726                       | 0.0971621                     | 0.175384                       |
| texture/wood_box_no_specular.xml     | 0.0772033                     | 0.151232                       | 0.0813137                     | 0.156096                       |
| texture/dragon/dragon_new_ply.xml    | 2.40284                       | 4.72215                        | 2.449                         | 4.68368                        |

![Variant vs Virtual](./assets/plot_variant_virtual_big.png)

As always, I removed the biggest case, Focusing Dragons in this case, and created a graph with less scale issues.

![Variant vs Virtual without Focusing Dragons](./assets/plot_variant_virtual.png)

And here is comparison between multisampled environment light testcases, the head case having increasing number of samples. I also tested with non-uniform sampling I used at first, and rejection sampling I right before submission.

| Testcase                | Init Time | Rendering Time (Non-Uniform Sampling) | Rendering Time (Rejection Sampling) (1 run) |
| ----------------------- | --------- | ------------------------------------- | ------------------------------------------- |
| head_env_light_1.xml    | 0.480417  | 0.0863338                             | 0.094445                                    |
| head_env_light_4.xml    | 0.482032  | 0.324461                              | 0.35339                                     |
| head_env_light_9.xml    | 0.480289  | 0.713553                              | 0.77416                                     |
| head_env_light_25.xml   | 0.478837  | 1.93651                               | 2.1619                                      |
| head_env_light_64.xml   | 0.48009   | 4.91133                               | 5.3379                                      |
| head_env_light_81.xml   | 0.478714  | 6.17004                               | 6.4833                                      |
| head_env_light_256.xml  | 0.479881  | 19.1987                               | 20.162                                      |
| head_env_light_400.xml  | 0.478432  | 29.877                                | 20.162                                      |
| head_env_light_625.xml  | 0.479133  | 46.5023                               | 49.161                                      |
| head_env_light_900.xml  | 0.480646  | 67.1025                               | 70.671                                      |
| audi-tt-glacier (1 run) | 4.0886    | 92.344                                | 92.504                                      |
| audi-tt-pisa (1 run)    | 1.0195    | 92.594                                | 100.36                                      |

For this, I created a line graph for Head case instead, for which no Audi case being used.

![Head Environment Light Time by Sample Count](./assets/plot_head_sampling.png)

I guess the non-linearity of the rejection sampling is actually caused by the lack of run samples, since it is run once instead of 10 times.

And finally, the new cases. Note that Audi cases differ considerably, the reason of which I have not been able to find. I might have used my computer a bit while rendering Pisa one, but I am not sure.

| Testcase                          | Initialization Time | Rendering Time |
| --------------------------------- | ------------------- | -------------- |
| cube_directional.xml              | 0.00107759          | 0.0486516      |
| cube_point_hdr.xml                | 0.00112352          | 0.0476622      |
| cube_point.xml                    | 0.00107655          | 0.0472609      |
| cube_spot.xml                     | 0.00111161          | 0.697898       |
| dragon_spot_light_msaa.xml        | 0.793014            | 12.9384        |
| empty_environment_latlong.xml     | 0.026575            | 0.00820551     |
| empty_environment_light_probe.xml | 0.0290373           | 0.00723625     |
| glass_sphere_env.xml              | 0.194476            | 0.0978511      |
| head_env_light.xml                | 0.488338            | 70.671         |
| mirror_sphere_env.xml             | 0.193074            | 0.0273269      |
| sphere_env_light.xml              | 0.345325            | 14.0511        |
| sphere_point_hdr_texture.xml      | 0.193863            | 0.0319999      |
| audi-tt-glacier (1 run)           | 3.7497              | 92.504         |
| auto-tt-pisa (1 run)              | 4.2037              | 100.36         |

![Times of New Cases](./assets/plot_hdr.png)

![Times of New Cases (Stacked)](./assets/plot_hdr_stacked.png)

As usual, here is the graph without Audi and head cases, which break the scale.

![Times of New Cases without Audis](./assets/plot_hdr_small.png)

![TImes of New Cases without Audis](./assets/plot_hdr_stacked_small.png)

## Conclusion

Yet another homework, and again something easy while being definitely non-trivial. I am yet again thankful for these homeworks, these homeworks actually teach me (and probably most of us) writing maintainable software in suitable deadlines much better than term projects. I do not feel that I am doing work in term project, it is unnecessarily too bureucratic while giving almost no benefit to the project itself. However, even though we are doing these homeworks alone, they teach me writing good software practices better. The AGILE approach emphasizes constant refactoring, and I do this in homeworks but not the term project, where we use AGILE as the official software engineering method. I think that developing such applications in such a long timespan (compared to the other homeworks we have done in the other courses) means that we are actually not writing alone, since the me two weeks later can actually be considered as a different person. The constant refactor and the implicit need for writing maintainable software is such an educating experience while being so entertaining!

[^1]: Yet another anime/manga/light novel reference! However, this time, it is from a less controversial one named [No Game No Life](https://anilist.co/manga/78399/No-Game-No-Life). The main characters (an elder brother named Sora and a little sister named Shiro) use the nickname 『　　』 in games, read as Kuuhaku. They both are pretty much geniuses in gaming, playing games almost the whole day without seeing a single ray of the Sun and having literally no single instance of defeat. They would then be summoned to another world where everything from thefts to wars between polities being done with games, ranging from backgammon to VR FPS games. The name is actually the combination of the characters' names, <ruby>空<rp>(</rp><rt>Sora</rt><rp>)</rp></ruby> and <ruby>白<rp>(</rp><rt>Shiro</rt><rp>)</rp></ruby>. When both of these Kanjis are concatenated, it becomes <ruby>空<rp>(</rp><rt>Kuu</rt><rp>)</rp></ruby><ruby>白<rp>(</rp><rt>haku</rt><rp>)</rp></ruby>, which means "blank" in Japanese. The brother's name means "empty, sky" and the sister's name means "white", so the name can roughly be translated as "whitespace", which is also another usage of this word, e.g. on keyboards.

---
{
  "author": "Erencan Ceyhan",
  "lang": "en",
  "tags": ["C++", "Programlama", "Teknoloji", "Gönderi", "Ödev", "Işın İzleme"]
}
---

---
{
   "date": "[2024](/gönderiler/2024)-[12](/gönderiler/2024/12)-29 23:39:24+03:00"
}
---

