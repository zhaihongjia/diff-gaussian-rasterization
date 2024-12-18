# Differential Gaussian Rasterization Improved

## NOTES

This repo is forked from [diff_gauss](https://github.com/dendenxu/diff-gaussian-rasterization) and used for our paper [SplatLoc](https://arxiv.org/abs/2409.14067)

The code changes are as follows:
- change the NUM_CHANNELS in `cuda_rasterizer/config.h` to support other dimensions.
- detach gradient backpropagation for features with channel greater than 3.


## Faster Backward Pass

This is only faster if there're large number of semi-transparent (almost) transparent Gaussians to be rendered since it might introduce some small overheads for regular rendering.

The original backward implementation uses `atomicAdd` on global CUDA memory.

We further accelerate this process by making use of the `__shared__` memory in a thread block to store the temporal accumulated gradients, just like the original did to the gaussian properties.

No api change is required for this functionality and you can directly check out what we changed in [backward.cu](cuda_rasterizer/backward.cu#417).

The change can be summarized in this pseudo-code:

```c++
__global__ void __launch_bounds__(BLOCK_X * BLOCK_Y)
renderCUDA(...) {

    __shared__ float3 s_dL_dmean2D[BLOCK_SIZE]; // allocated shared memory
    s_dL_dmean2D[block.thread_rank()].x = 0.0f; // fill shared memory with zeros

    for (int j = 0; !done && j < min(BLOCK_SIZE, toDo); j++) { // iterate over gaussian that has a influence on this pixel
        // Compute gradients
        ...

        // Update gradients w.r.t. 2D mean position of the Gaussian
        atomicAdd(&s_dL_dmean2D[j].x, dL_dG * dG_ddelx * ddelx_dx);
        atomicAdd(&s_dL_dmean2D[j].y, dL_dG * dG_ddely * ddely_dy);
    }

    atomicAdd(&dL_dmean2D[global_id].x, s_dL_dmean2D[block.thread_rank()].x);
    atomicAdd(&dL_dmean2D[global_id].y, s_dL_dmean2D[block.thread_rank()].y);
}
```

In an effort to make this process even faster, we've also implemented a warp-reduction based version of the backward pass on top of the `__shared__` memory optimization.

By directly communicating the gradient accumulation in a 32-thread warp using:

```c++
__device__ float warpReduceSum(float value) {
    auto warp = cg::coalesced_threads();
    for (int offset = warp.size() / 2; offset > 0; offset /= 2) {
        value += warp.shfl_down(value, offset);
    }
    return value;
}
```

And later aggregate the warp sum into `__shared__` memory:

```c++
...
			// Use a single thread from each warp to perform block level reduction
			if (block.thread_rank() % warp.size() == 0) {
				for (int ch = 0; ch < C; ch++) {
					atomicAdd(&(s_dL_dcolors[ch * BLOCK_SIZE + j]), w_dL_dcolors[ch]);
				}
				atomicAdd(&(s_dL_ddepths[j]), w_dL_ddepths);
				atomicAdd(&s_dL_dmean2D[j].x, w_dL_dmean2D.x);
				atomicAdd(&s_dL_dmean2D[j].y, w_dL_dmean2D.y);
				atomicAdd(&s_dL_dconic2D[j].x, w_dL_dconic2D.x);
				atomicAdd(&s_dL_dconic2D[j].y, w_dL_dconic2D.y);
				atomicAdd(&s_dL_dconic2D[j].w, w_dL_dconic2D.w);
				atomicAdd(&(s_dL_dopacity[j]), w_dL_dopacity);
			}
...
```

We can shave off another 2-3ms for the backward pass at the start of the training, but curiously it couldn't persist during the whole training process.

Thus by default only the `__shared__` memory optimization is enabled and in use.

Note: this seems slower... See: https://developer.nvidia.com/blog/gpu-pro-tip-fast-histograms-using-shared-atomics-maxwell

## Tile-Based Culling

Using the method mentioned: [StopThePop: Sorted Gaussian Splatting for View-Consistent Real-time Rendering](https://github.com/r4dl/StopThePop-Rasterization), we borrow the tile-based culling scheme here to reduce the computational cost during training and rendering.

This section of code is directly adapted from their repository.

```c++
...
    constexpr float alpha_threshold = 1.0f / 255.0f;
    const float opacity_power_threshold = log(conic_opacity[idx].w / alpha_threshold);
    glm::vec2 max_pos;
    const glm::vec2 tile_min = {x * BLOCK_X, y * BLOCK_Y};
    const glm::vec2 tile_max = {(x + 1) * BLOCK_X - 1, (y + 1) * BLOCK_Y - 1};
    float max_opac_factor = max_contrib_power_rect_gaussian_float<BLOCK_X-1, BLOCK_Y-1>(conic_opacity[idx], points_xy[idx], tile_min, tile_max, max_pos);

    if (max_opac_factor > opacity_power_threshold) {
        continue;
    }
...
```

Note: this seems slower...

## Tile-Mask Rendering

**Note: this api hasn't been fully tested yet.**

We additionaly provide a interface for adding a tile-mask to the gaussian rasterizer.

Turns out the tile-based rendering rasterization pipeline can be easily masked out to provide a patch-like rendering result (to simulate a NeRF-like ray sampling approach).

To implement this as efficiently as possible, we:

1. Mark points that's not to be rendered as early as possible in the `preprocessCUDA` kernel.
2. Make all subsequent operations faster by not including masked-out tiles in the sorting and `renderCUDA` kernel.

The tile mask can be defined as:

```python
from diff_gauss import GaussianRasterizationSettings, GaussianRasterizer
raster_settings = GaussianRasterizationSettings(...)
rasterizer = GaussianRasterizer(raster_settings=raster_settings)

BLOCK_X, BLOCK_Y = 16, 16
tile_height, tile_width = (raster_settings.image_height + BLOCK_Y - 1) // BLOCK_Y, (raster_settings.image_width + BLOCK_X - 1) // BLOCK_X
tile_mask = torch.ones((tile_height, tile_width), dtype=torch.bool, device='cuda')

rendered_image, rendered_depth, rendered_alpha, radii = rasterizer(
    means3D = means3D,
    means2D = means2D,
    shs = shs,
    colors_precomp = colors_precomp,
    opacities = opacity,
    scales = scales,
    rotations = rotations,
    cov3D_precomp = cov3D_precomp,
    tile_mask = tile_mask,
)
```

## Fixed `ImageState` Buffer Size

In the [original implementation](https://github.com/graphdeco-inria/diff-gaussian-rasterization), the size of the `ranges` member of the struct `ImageState` was too large (same as the number of pixels).

In reality, only `number of tiles` of `ranges` are needed, as the `ranges` are used to store the start and end indices of the gaussian splats in the `GeometryState` buffer.

We fix this by simply replacing the memory allocation of `ImageState` with:

```c++
CudaRasterizer::ImageState CudaRasterizer::ImageState::fromChunk(char*& chunk, size_t N, size_t M)
{
	ImageState img;
	obtain(chunk, img.n_contrib, N, 128);
	obtain(chunk, img.ranges, M, 128);
	return img;
}
```

## Fixed Culling

The [original repository](https://github.com/graphdeco-inria/diff-gaussian-rasterization)'s implementation for view-space culling wasn't effective (no points were culled).

We fixed that with an improved OpenGL like culling function:

```c++
__forceinline__ __device__ bool in_frustum(int idx,
	const float* orig_points,
	const float* viewmatrix,
	const float* projmatrix,
	bool prefiltered,
	float3& p_view, // reference
	const float padding = 0.01f, // padding in ndc space // TODO: add api for changing this
	const float xy_padding = 0.5f // padding in ndc space // TODO: add api for changing this
	)
{
	float3 p_orig = { orig_points[3 * idx], orig_points[3 * idx + 1], orig_points[3 * idx + 2] };
	p_view = transformPoint4x3(p_orig, viewmatrix); // write this outside
	if (prefiltered) return true;

	// Bring points to screen space
	float4 p_hom = transformPoint4x4(p_orig, projmatrix);
	float p_w = 1.0f / (p_hom.w + 0.0000001f);
	float3 p_proj = { p_hom.x * p_w, p_hom.y * p_w, p_hom.z * p_w };

	return (p_proj.z > -1 - padding) && (p_proj.z < 1 + padding) && (p_proj.x > -1 - xy_padding) && (p_proj.x < 1. + xy_padding) && (p_proj.y > -1 - xy_padding) && (p_proj.y < 1. + xy_padding);
}
```

## Depth & Alpha Backward

**Note: this functionality is directly copied from the [slothfulxtx repository](https://github.com/slothfulxtx/diff-gaussian-rasterization).**

Except for the RGB image, we also support render depth map and alpha map (both forward and backward process) compared with the [original repository](https://github.com/graphdeco-inria/diff-gaussian-rasterization).

We modify the dependency name as **diff_gauss** to avoid dependecy conflict with the original version. You can install our repo by executing the following command lines

Here's an example of our modified differential gaussian rasterization repo
```python
from diff_gauss import GaussianRasterizationSettings, GaussianRasterizer
raster_settings = GaussianRasterizationSettings(...)
rasterizer = GaussianRasterizer(raster_settings=raster_settings)

rendered_image, rendered_depth, rendered_alpha, radii = rasterizer(
    means3D = means3D,
    means2D = means2D,
    shs = shs,
    colors_precomp = colors_precomp,
    opacities = opacity,
    scales = scales,
    rotations = rotations,
    cov3D_precomp = cov3D_precomp
)
```

Details: By default, the depth is calculated as 'median depth', where the depth values of each pixels covered by 3D Gaussian Splatting are set to be the depth of the 3D Gaussian center. Thus, there exist numerical errors when the scales of 3D Gaussian are large. However, thanks to the densificaiton scheme, most 3D Gaussians are small. Currently, we ignore the numerical error of depth maps. 

## Differential Gaussian Rasterization

**Note: this is the original readme for the [original diff-gaussian-rasterization repository](https://github.com/graphdeco-inria/diff-gaussian-rasterization).**

Used as the rasterization engine for the paper "3D Gaussian Splatting for Real-Time Rendering of Radiance Fields". If you can make use of it in your own research, please be so kind to cite us.

<section class="section" id="BibTeX">
  <div class="container is-max-desktop content">
    <h2 class="title">BibTeX</h2>
    <pre><code>@Article{kerbl3Dgaussians,
      author       = {Kerbl, Bernhard and Kopanas, Georgios and Leimk{\"u}hler, Thomas and Drettakis, George},
      title        = {3D Gaussian Splatting for Real-Time Radiance Field Rendering},
      journal      = {ACM Transactions on Graphics},
      number       = {4},
      volume       = {42},
      month        = {July},
      year         = {2023},
      url          = {https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/}
}</code></pre>
  </div>
</section>
