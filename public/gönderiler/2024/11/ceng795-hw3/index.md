# CENG795-HW3: Distributed Ray Tracing

Hello, minasan! Welcome to my blog post for the third assignment. The weather was somewhat clear in the previous homework, while I was gloomy. This homework, it seems like weather and I swapped roles. I managed to complete the homework while also finishing the remnants from the previous one. Meanwhile, we now have snow and a gloomy weather in Ankara.

The homework was suprisingly easy. I gave 3 full days to complete it, but I finished before the half of the second day. I could have finished even earlier but I tried to fix the intersection problem, which shows itself as gaps in some meshes, especially the Chinese dragon mesh, which has been used for at least one test case in each homework (`chinese_dragon`, `marching_dragons` and `focusing_dragons`). Unfortunately, I did not succeed to fix this bug. I however realized that those black points are not the background, but instead ambient light. For some reason, the ray correctly intersects with the mesh but the light ray intersects with something even though it should not. I will try to solve it again for the next homework, but it seems that I need some sort of revelation to do it, it should be an easy problem but all the fixes I thought failed.

The other parts, the parts which we need to successfully finish the homework, are done correctly, with a small problem I just forgot to fix before submission.

## Implementation

Every single feature of this homework depend on the multisampling. We were required to implement several features, but at first, I needed to implement sampling. Since Halton or Hammerslay sampling techniques improve the result marginally while being much harder to implement, I just went with the main sampling method we needed to do: jittered sampling. For the rays, I used Mersenne Twister, which is implemented in the C++ STL as `std::mt19937`. However, it needed that context and I was too lazy to pass the context down to the ray tracing code, so I used `rand`, the C stdlib function. I know that it is much worse compared to `std::mt19937` in terms of the randomization quality; however, I think it is pretty enough. Except for the jittered sampling, I did not use Mersenne Twiser, yet the results were sufficient. The main problem I actually feared was thread safety, since neither of them are inherently thread-safe, but `std::mt19937` contains its own context, so that I can create RNG state per thread and use this state. As I said, I was too lazy, so I opted to `rand`. I have not have a problem with this for some reason. Maybe it is due to the fact that the race condition would only assign the RNG state to a previous one, which merely decreases randomness. [^1]

### Multisampling

My ray creation code now checks for the sample size. If it is 1 (no multisampling), it send the ray through the center of the pixel, just like how it was done before. I added that part since the old scenes gained some artifacts due to the fact that 1 sample means uniform sampling with 1 instance, which results in weirdness.

![Cornellbox with 1 sample](images/cornellbox_wobble.png)

If the sample size is not 1, the ray creator creates random numbers, between the range `[offset, offset + step)`, where `offset` is the top left corner of the sample (`sample_pos * step`) and `step` is the size of the area of each sample (`1 / numSamples`). This directly creates a random number that can be used as the offset from the top left of the pixel, so that I can put it in place of `0.5`, which I used as to center the ray's target inside the pixel.

Yep, this is just the sampling implementation. To make this multi, I simply added two inner loops, took the sum of the pixel values from each sample and divided by the `numSamples`. This is actually a pretty bad way to combine samples, I need a Gaussian filter, but the output is pretty nice, so I don't think there is a severe problem. This may probably be an issue for higher frequency scenes with more samples, but for the current testcases, it is enough.

I changed the `other_dragon` to have 16 samples, and oh boy, it looks awesome.

![Other Dragon with 16 Samples](images/other_dragon_16_samples.png)

### Roughness

Since both DOF and area lights need different sampling, I wanted to implement an easier feature and roughness looked such. The actual mathematics behind the ONB creation is a bit harder, but the result is just a simple function, which I also used in area lights:

```cpp
void get_uv(glm::vec3 n, glm::vec3& u, glm::vec3& v) {
    glm::vec3 n_prime = n;
    glm::vec3 n_abs = glm::abs(n);
    if (n_abs.x < n_abs.y) {
        if (n_abs.x < n_abs.z) {
            n_prime.x = 1.0f;
        } else {
            n_prime.z = 1.0f;
        }
    } else {
        if (n_abs.y < n_abs.z) {
            n_prime.y = 1.0f;
        } else {
            n_prime.z = 1.0f;
        }
    }

    u = glm::normalize(glm::cross(n, n_prime));
    v = glm::normalize(glm::cross(n, u));
}
```

After creating this function (which is just a bunch of `if`s followed by some normalized cross products), implementing roughness was a piece of cake.

```cpp
glm::vec3 u, v;
if (mat.roughness != 0) {
    get_uv(ray.direction, u, v);
    float chi_1 = ((float)std::rand() / RAND_MAX) - 0.5f;
    float chi_2 = ((float)std::rand() / RAND_MAX) - 0.5f;
    ray.direction = glm::normalize(ray.direction + mat.roughness * (chi_1 * u + chi_2 * v));
}
```

The `(float)std::rand() / RAND_MAX` is used everywhere where I used `rand`. Creating a macro would probably be a wise choice, but I will replace them with `std::mt19937` anyways.

The result is, well, not correct at first. The code I put here is the final code, and it works.

![Cornellbox Roughness](images/cornellbox_brushed_metal_correct.png)

However, at first, the result was pretty weird.

![Cornellbox Roughness with Weird Sections](images/cornellbox_brushed_metal_sphere_sections.png)

The reason was actually simple: the random number was in the range `[0, 1)` at first, which created this weird pattern. When I added `- 0.5f` parts you can see in the code, it was fixed.

However, there is still a small problem, the outer parts of the orange sphere looks a bit darker compared to the reference image. In the following image, top half is the reference image and bottom half is my result.

![Cornellbox Roughness Comparison](images/brushed_combined.png)

Apart from this small bug, roughness was successfull. I don't know why this bug happens, and it did not happen in other glossy surface, the mirror. I sense that my conductor implementation has a tiny problem, since `other_dragon` also has a small visual difference compared with the reference image.

### Depth of Field

After finishing roughness, I continued with DOF. For DOF, I needed to sample a point on the lens. It was suprisingly easy. As in multisampling, I only needed to change the ray creation function. If the `apertureSize` is not `0` (which denotes a perfect pinhole camera, and also the default value if `apertureSize` is not detected), the lens sampling part is run. After applying the formula written in the `distribution_ray_tracing_explained.pdf`, I got correct results. Running your code for the first time after a new feature and seeing that the code runs correctly will always be one of the biggest satisfications I can get from programming. The only issue I saw was that the result was a bit sandy. However, I now know that the reason is due to the fact that I used uniform sampling while sampling the lens. The problem here is that, I realized that the sandyness of both area lights and DOF is caused by the same thing. After learning this information from the instructor, I applied the solution for the area lights, but not to DOF. Hence, my submission creates sandy DOF blur in its current form. I quickly implemented the solution in a few minutes while writing the post.

![Sandy Sphere DOF](images/spheres_dof_100_sample.png)

![Non-Sandy Sphere DOF **\[NYI\]**](images/spheres_dof.png)

### Area Light

As with the previous features, implementing the area lights was easy. I probably spent more time to implement the parser (which I now do using my spinal cord instead of my brain) compared to implementing the actual feature. I applied the formula in the `areaLight.pdf`, and BOOM, I now have area light, or is it? There were two main bugs.

![First Try of Cornellbox Area](images/cornellbox_area_fever_dream.png)

Due to the ripple-like circles, I guessed that the problem was in the irradiance calculation, and I was right. I had forgotten to divide by $r^2$ while calculating the irradiance. After fixing it, the result was much better:

![Second Try of Cornellbox Area](images/cornellbox_area_more_fever_dream.png)

But this is still bad! Finding the cause of this problem was a bit trickier than the other, but I found out that the $w_i$ in the PDF and my calculation had the opposite signs. Putting a `-` when calculating the dot product solved the problem.

![Third Try of Cornellbox Area](images/cornellbox_area_nomax.png)

Hmm, the spheres and planes in bottom were correct but the output had a noise around the light, and around the top stitch points of red and blue planes. After some painful debugging sessions, I found out that the dot product became negative in some pixels. I then clamped the dot product to `0` from below, like in our diffuse and specular shading implementations. And, voilà!

![Correct Cornellbox Area](images/cornellbox_area_100_sample.png)

It was a bit sandy tho. The solution for this is actually in my submission, and the result was nice. However, I realized that shuffling the jitter regions was not necessary for good output in this case, the output was still nice.

![Correct Non-Sandy Cornellbox Area with Non-Shuffled Areas](images/cornellbox_area_jitter_with_connection.png)

For comparison, here is the result when I shuffled the jitter regions:

![Correct Non-Sandy Cornellbox Area](images/cornellbox_area_correct.png)

### Motion Blur

Finally, the last feature! Implementing it should also be easy, right? Well, the main testcase for this, `cornellbox_boxes_dynamic` results in this:

![Cornellbox Boxes Dynamic](images/cornellbox_boxes_dynamic_error.png)

No matter what I did, I was unsuccessfull to bring this testcase up. However, of course, the hope is not lost! Motion blur has a very simple addition to the XML, just a simple `MotionBlur` tag with the `xyz` components of the translation. Hence, I just added it to a scene that works with my raytracer. This means that, unfortunately, the motion blur examples will not be `cornellbox_boxes_dynamic`. I mainly used `cornellbox_area`. Here is `cornellbox_area` with the yellow sphere having motion blur `0 0 3`:

![Cornellbox Area with Motion Blur](images/cornellbox_area_motion_blur_003.png)

And here is the same scene with motion blur `0 2 3`:

![Cornellbox Area with Motion Blur in 2 Axes](images/cornellbox_area_motion_blur_023.png)

Of course the quality of the image degrades when the motion blur strength is increased. I tried to do jittered sampling for `t`, with the same shuffled result used for area light and DOF. The result is, interesting.

![Cornellbox Area with Motion Blur and Jittered Sampling **\[NYI\]**](images/cornellbox_area_motion_blur_jitter.png)

There is less noise, but there are bands in the shadow. I managed to decrease the banding (which is probably caused by the fact that I used the same shuffle for both area light and motion blur) by using a different shuffled area.

![Cornellbox Area with Motion Blur and Better Jittered Sampling **\[NYI\]**](images/cornellbox_area_motion_blur_better_jitter.png)

## Results

![Focusing Dragons](images/focusing_dragons.png)

![Dragon Dynamic](images/dragon_dynamic.png)

![Metal Glass Plates](images/metal_glass_plates_forgot.png)

Do you see the problem? Yep, the glasses do not have roughness since I forgot to implement roughness for dielectric materials before submission. After three applications of copy-paste-fu, I got the correct result:

![Correct Metal Glass Plates **\[NYI\]**](images/metal_glass_plates_correct.png)

## Performance

In this homework, I started to use happly to parse PLY files, since my ad-hoc parser full of pointer hacks was not scalable for different kinds of PLY files in the testcases. The problem is, well, happly is slow. This increased the initialization times.

In the table, the number between the parantheses represents the number of samples. Just like in the previous homework, all entries are actually the geometric mean of 10 runs.

| Testcase                       | Initialization Time | Rendering Time | Rendering Time / Samples |
| ------------------------------ | ------------------- | -------------- | ------------------------ |
| other_dragon (1)               | 1.9006              | 0.3248         | 0.3248                   |
| other_dragon (4)               | 1.9                 | 1.1906         | 0.2977                   |
| other_dragon (9)               | 1.8981              | 2.5243         | 0.2805                   |
| other_dragon (16)              | 1.8961              | 4.3362         | 0.2710                   |
| cornellbox_area (100)          | 0.0009              | 6.5293         | 0.0653                   |
| cornellbox_brushed_metal (400) | 0.0008              | 32.3397        | 0.0808                   |
| dragon_dynamic (100) (1 run)   | 1.9166              | 283.14         | 2.8314                   |
| focusing_dragons (100)         | 0.7449              | 49.2412        | 0.4924                   |
| metal_glass_plates (36)        | 0.0009              | 16.0495        | 0.4458                   |
| spheres_dof (100)              | 0.0007              | 3.7865         | 0.0378                   |

As we can see with `other_dragon`, the rendering time per sample is more-or-less the same. It even decreases a bit as we increase the number of samples. Interesting, isn't it?

And of course, here are the plots.

![Performance Plot with Dragon Dynamic (Separate)](images/plot_separate_dragon.png)

![Performance Plot with Dragon Dynamic (Stacked)](images/plot_stacked_dragon.png)

The `dragon_dynamic` case is so long that it breaks the scale of the graph, so I also created graphs without it.

![Performance Plot (Separate)](images/plot_separate.png)

![Performance Plot (Stacked)](images/plot_stacked.png)

## Conclusion

If the previous homework had made me question my technical skills, this homework made me question my memory and attention. Apart from the small gloss problem and light intersection problem, every other problem that remained in my submission can be solved easily, but I just forgot to do them.

I also think about the difference between the difficulties of the homeworks. The previous one was hard, this one was easy, and the next one will probably be also hard. It is pretty interesting, right?

[^1]: While writing the post, I searched for the thread-safety of `rand`, and even though it is not thread-safe in the standard as I remember correctly, it seems like the Windows implementation is thread-safe.

---
author: Erencan Ceyhan
lang: en
tags:
- C++
- Programlama
- Teknoloji
- Gönderi
- Ödev
- Işın İzleme
date: 2024-11-29 18:09:52+03:00
---
