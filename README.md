![](man/figures/diffuseRlogo.png)

# diffuseR

[![CRAN status](https://www.r-pkg.org/badges/version/diffuseR)](https://CRAN.R-project.org/package=diffuseR)
[![Lifecycle: experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://lifecycle.r-lib.org/articles/stages.html#experimental)

## Overview

`diffuseR` is a functional R implementation of diffusion models, inspired by Hugging Face's Python `diffusers` library. The package provides a simple, idiomatic R interface to state-of-the-art generative AI models for image generation and manipulation using base R and the `torch` package. No Python dependencies. Currently supports Windows and Linux cpu and cuda devices.

```r
# Simple text-to-image generation
library(diffuseR)
image <- txt2img(prompt = "A serene landscape with mountains and a lake at sunset",
                 model = "sd21", # Specify the model to use, e.g., "sd21" for Stable Diffusion 2.1
                 save_to = "landscape.png")
```

## Example output

![](man/figures/20250528_165522_Calvin_and_Hobbes_on_a_beach__Calvin_wearing_a_red.png)
![](man/figures/20250528_200344_Calvin_and_Hobbes_on_a_beach__Calvin_wearing_a_red.png)
![](man/figures/20250529_000111_A_majestic_red_fox_standing_on_a_moss_covered_log_.png)
![](man/figures/20250529_002833_Hulk_Hogan_standing_on_a_moss_covered_log_in_an_an.png)

## Installation

First install torch. As per [this comment](https://github.com/mlverse/torch/issues/1198#issuecomment-2419363312), using the pre-built binaries from [https://torch.mlverse.org/docs/articles/installation#pre-built](https://torch.mlverse.org/docs/articles/installation#pre-built) is heavily recommend. "The pre-built binaries bundle the necessary CUDA and cudnn versions, so you don't need a global compatible system version of CUDA":

```r
options(timeout = 600) # increasing timeout is recommended since we will be downloading a 2GB file.
# For Windows and Linux: "cpu", "cu124" are the only currently supported
# For MacOS the supported torch versions are: "cpu-intel" or "cpu-m1"
kind <- "cu124"
version <- available.packages()["torch","Version"]
options(repos = c(
  torch = sprintf("https://torch-cdn.mlverse.org/packages/%s/%s/", kind, version),
  CRAN = "https://cloud.r-project.org" # or any other from which you want to install the other R dependencies.
))
install.packages("torch")
```

You can install the development version of diffuseR from GitHub:

```r
# install.packages("devtools")
devtools::install_github("cornball-ai/diffuseR")
# Or
# install.packages("targets")
targets::install_github("cornball-ai/diffuseR")
```

## Features

- **Text-to-Image Generation**: Create images from textual descriptions
- **Image-to-Image Generation**: Modify existing images based on text prompts
- **Two Models**: Support for Stable Diffusion 2.1 and SDXL (more coming soon)
- **Scheduler Options**: DDIM (more coming soon)
- **Device Support**: Works on both CPU and GPU
- **R-native Interface**: Functional programming approach that feels natural in R

## Quick Start

### Basic Usage

**Warning**: The first time you run the code below, it will download ~7GB of model files [from here](https://huggingface.co/cornball-ai/sd21-R/tree/main) and load them into memory. Ensure you have enough RAM, disk space, and a stable internet connection. Memory management with deep learning models is crucial, so consider using a machine with sufficient resources; ~8GB of free RAM is recommended for running Stable Diffusion 2.1 on CPU only.

```r
library(diffuseR)
torch::local_no_grad()

# Generate an image from text
cat_img <- txt2img(
  prompt = "a photorealistic cat wearing sunglasses",
  model = "sd21", # Specify the model to use, e.g., "sd21" for Stable Diffusion 2.1
  download_models = TRUE, # Automatically download the model if not already present
  steps = 30,
  save_to = "cat.png",
)

torch::cuda_empty_cache()
```

### Advanced Usage with GPU

The unet is the most computationally-intensive part of the model, so it is recommended to run it on a GPU if possible. The decoder and text encoder can be run on CPU if you have limited GPU memory. SDXL's unet requires a minimum of 6GB of GPU memory, while Stable Diffusion 2.1 requires a minimum of 2GB.

```r
# Increasing timeout is recommended since we will be downloading 5.1 and 2.8GB model files, among others.
options(timeout = 900) 

# Downlaod a test image
url <- "https://upload.wikimedia.org/wikipedia/commons/4/47/PNG_transparency_demonstration_1.png"
utils::download.file(url, "test.png", mode = "wb")

library(diffuseR)
torch::local_no_grad() # Prevents torch from tracking gradients, which is not needed for inference

# Assign the various deep learning models to devices
model_name = "sdxl"
input_image = "test.png"
devices = list(unet = "cuda", decoder = "cpu",
               text_encoder = "cpu", encoder = "cpu")

m2d <- models2devices(model_name = model_name, devices = devices,
                      unet_dtype_str = unet_dtype_str, download_models = TRUE)

pipeline <- load_pipeline(model_name = model_name, m2d = m2d, i2i = TRUE,
                          unet_dtype_str = unet_dtype_str)

# Generate an image from text
cat_img <- txt2img(
  prompt = "a photorealistic cat wearing sunglasses",
  model_name = model_name,
  steps = 30,
  save_to = "cat2.png",
)

gambling_cat <- img2img(
  input_image = "test.png",
  prompt = "a photorealistic cat wearing sunglasses and gambling",
  model_name = model_name,
  devices = devices,
  pipeline = pipeline,
  unet_dtype_str = unet_dtype_str,
  num_inference_steps = 30,
  strength = 0.75,
  guidance_scale = 7.5,
  save_file = TRUE, # Save the generated image
  filename = "gambling_cat.png"
)

# Clear out pipeline to free up GPU memory
pipeline <- NULL
torch::cuda_empty_cache()
```

## Supported Models

Currently supported models:

- Stable Diffusion 2.1
- Stable Diffusion XL (SDXL)
- More coming soon!

## Roadmap

Future plans for diffuseR include:

- [ ] Inpainting support
- [ ] Additional schedulers (PNDM, DPMSolverMultistep, Euler ancestral)
- [ ] text-to-video generation

## How It Works

diffuseR uses TorchScript models exported from PyTorch implementations for the deep learning parts of the implementation. This approach was the quickest and easiest way to build the machinery that supports the deep learning models in stable diffusion. Full R torch implementations of the deep learning models are planned for the future, but this initial version allows users to quickly get started with diffusion models in R without needing to rely on any Python.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the Apache 2. License - see the LICENSE file for details.

## Acknowledgments

- Hugging Face for the original diffusers library
- Stability AI for Stable Diffusion
- The R and torch communities for their excellent tooling
