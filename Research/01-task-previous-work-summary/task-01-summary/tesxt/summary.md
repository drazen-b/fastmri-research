<style>
  body {
    font-size: 14px;
    line-height: 1.3;
  }
</style>

# MRI image reconstruction using fastMRI library

## fastMRI

fastMRI is a collaborative exploratory project by Facebook AI Research with the goal of developing faster ways of MRI image acquisition. It consists of datasets of brain and knee MRI images, and of code repository with tools to work with dataset and model implementation.

## Models in fastMRI library

### Zero-Filled

The zero-fill method inserts zeros in the places of the uncollected k-space data points. It expands the undersampled k-space data to the size needed for a full-resolution image, but with zeros where we lack actual data. After zero-filling the k-space inverse Fourier transform is applied to convert data back to the spatial domain.

### Compressed Sensing

Compressed Sensing is based on a mathematical principal which states that images and signals can be represented with fewer data without losing a significant amount of information. The implementation in fastMRI uses BART (The Berkeley Advanced Reconstruction Toolbox) which is a free and open-source image-reconstruction framework for Computational Magnetic Resonance Imaging. 

### U-Net

<!-- U-Net model provided by fastMRI is meant to be used in the reconstruction of images taken with single coil. To use it on images taken with multiple coils it's first needed to do zero-fill method on each coil image.
Model consists of two deep convolution network paths. First (left on image) is compression path and second (right on image) is decompression path. 
Compression path consists of 3x3 convolution blocks, each convolution is followed with instance normalisation and ReLU activation function. Blocks are down sampled using max-pooling with step of two.
Decompression path consists of similar blocks, but blocks are up scaled with each step.
Compression and decompression paths are connected with skip connections.
The path of compression allows capturing the context and reveals what's in the image, while the path of decompression reveals where is it located in the image. To improve localization, high-resolution features from the compression path are connected with the outputs from the up-sampling path through skip connections. With the information from the skip connections, a more precise output from the block is obtained. At the end of the up-sampling path, 1x1 convolutions are used to reduce the number of channels to one without changing the spatial resolution. To predict the edge pixels of the input image, a mirroring technique is applied to fill in the missing data. -->

The U-Net model provided by fastMRI is designed for single-coil image reconstruction but can be adapted to multi-coil images using the zero-fill method for each coil. The model consists of two main paths: a encoder path and decoder path. Encoder paths goal is context and content capture using 3x3 convolutions, instance normalization, ReLU activation and max-pooling down-sampling. Decoder path is used for spatial localization and block up-scaleing. Both paths are connected with skip connections which enchance detail and precision in the output using high-resolution features.

<div align="center">
    <img src="./images/unet-architecture.png" width="400">
</div>

### End-to-End VarNet

End-to-End Varnet model is designed to learn complete process of reconstruction. End-to-End means we can give it raw data without any preprocessing and it will give out processed result.

It takse multi coil k-space as input and applies a series of refinement steps which we call cascades. At the begginning sensitivity maps are estimated using SME module. In each cascade Data Consistency module (DC) and Refinement module (R) is applied. After cascades, IFT is performed to get image from each k-space, followed by root-sum-squares reduction (RSS) for each pixel. Image is then cropped and output is given.


#### SME module

Sensitivity Map Estimation module estimates sensitivity maps which are later used in Refinement module. In traditional VarNet sensitivity maps are computed using the ESPIRiT algorithm, but here they're computed using SME module:

<!-- $$ H = dSS \circ CNN \circ \mathcal{F}^{-1} \circ M_{\text{center}} $$ -->

<div align="center">
    <img src="./images/sme-module.png">
</div>

 - M<sub>center</sub> zeroes all lines except ACS lines. ACS are autocalibration lines, central region of k-space corresponding to low frequencies which is used for sensitivity estimation.
- CNN is convolutional neural network. Same as one used in cascades (U-Net), except with fewer channels and fewer parameters.
- dSS normalizes sensitivity maps. They need to satisfy equiation below:

<!-- $$ \sum_{i=1}^{N} S_i^* S_i = 1 $$ -->

<div align="center">
    <img src="./images/normalise-equation.png">
</div>

#### DC module

Data Consistency computes correction map which brings the intermediate k-space closer to the measured k-space values. Ut performs subtraction between inputs and identifies differences between them. After that, it applies a correction map to differences identified in the previous step. Adjusting intermediate k-space so it becomes more consistent with measured k-space values.

#### Refinement module

The refinement module maps multi-coil k-space data into one image. It takes intermediate k-space and estimated sensitivity maps (ESM) as input. Applies IFT to intermediate k-space, then using ESM it reduces it to a single image. It applies U-Net on the image and then, using ESM, expands them to images seen by each coil. After all of those steps, it applies FT so output is k-space.

<div align="center">
    <img src="./images/e2e-varnet-architecture.png" width="400">
</div>

## Results:

<div align="center">
  <img src="./images/test-images.png" width="400">
  <br>
  <em>Test images</em>
</div>

<div align="center">
  <div>
    <span style="display: inline-block; text-align: center; margin-right: 20px; margin-top: 20px;">
      <img src="./images/zero-fill.png" width="300">
      <br>
      <em>Zero-fill</em>
    </span>
    <span style="display: inline-block; text-align: center; margin-top: 20px;">
      <img src="./images/compressed-sensing-images.png" width="300">
      <br>
      <em>Compressed Sensing</em>
    </span>
  </div>
  <div>
    <span style="display: inline-block; text-align: center; margin-right: 20px; margin-top: 20px;">
      <img src="./images/unet-images.png" width="300">
      <br>
      <em>U-Net</em>
    </span>
    <span style="display: inline-block; text-align: center; margin-top: 20px;">
      <img src="./images/varnet-images.png" width="300">
      <br>
      <em>End-to-End VarNet</em>
    </span>
  </div>
</div>

## Current state-of-art

For a long time E2E VarNet presented best results on fastMRI bechmark, but in March 2023 HUMUS-Net network which outperforms E2E VarNet has been presented. It's an unrolled, Transformer-convolutional hybrid network for accelerated MRI reconstruction. Architecture consists of a sequence of sub-networks (cascades). Each cascade represents an unrolled iteration of an underlying optimization algorithm in k-space with an image-domain denoiser, the HUMUS-Block. The HUMUS-Block acts as an image-space denoiser that receives an intermediate reconstruction from the previous cascade and performs a single step of denoising to produce an improved reconstruction for the next cascade. It extracts high-resolution, shallow features and low-resolution, deep features through a novel multi-scale transformer-based block, and synthesizes high-resolution features from those. Limitation of current architecture is that it required fixed-size inputs.

<div align="center">
  <img src="./images/humus-comparison.png" width="400">
  <br>
  <em>Humus results comparison</em>
</div>

<div align="center">
  <img src="./images/humus-perf.png" width="400">
  <br>
  <em>Humus performance comparison</em>
</div>

## Conclusion

FastMRI library provides set of tools, models and dataset needed to work on MRI reconstruction. Techniques it provides are zero-fill, compressed sensing, U-Net and E2E VarNet. For a long time E2E VarNet presented best results, until HUMUS-Net, which brought better results, was presented in March 2023. But still, reconstructions are still now clinicaly valid because of their total lack of details. Even if they were, they would need a approval of medical experts and couldn't be used in real diagnostics.

## Master thesis

Goal is to implement modified U-Net and VarNet models and adapt them for the task of reconstructing MRI brain images. The display, explain and compare the results and determine the accuracy of the developed system.

## Literature

