# Reconstruction-Of-Nuclear-Medicine-Imaging-Systems-Simulated-Acquisition-Scintigraphy-SPECT

Steps to exaplain:

1) Create the castor system file (geo)
2) Create the histogram castor data from the root file
3) Reconstruct the dataset, go over options
4) How to read castor files in Matlab, ImageJ
5) Provide example projections and reconstruction


## 3. Reconstruction in CASToR

CASToR provides a tool to create a CASToR datafile directly from a GATE macro and root file: `castor-GATErootToCastor`. To use: 

```castor-GATErootToCastor -i path/to/ifile.root -o path/to/outfile -m path/to/macrofile.mac -s scanner_alias -sp_bins bins_x,bins_y``` <br />
where <br />
`path/to/ifile.root` is the root output file from Gate <br />
`path/to/outfile` is the base name to save the CASToR datafile to <br />
`path/to/macrofile.mac` is the Gate macro file used to generate the root file <br />
`scanner_alias` corresponds to a `scanner_alias.geom` file in your `castor/config/scanner/` directory.  <br />
`bins_x,bins_y` are the transaxial and axial number of bins for projections, separated by a comma.

Note that CASToR expects the macro to have units of cm. Comments after commands can also cause issues so make sure all comments are on their own new line. 

The `scanner_alias.geom` file defines the components of your detector. Below is an example for a SPECT system: 

```ruby
modality: SPECT_CONVERGENT
scanner name: SPECT_BRIGHTVIEW
description: This scanner description is based on an actual scanner, however, this implementation is not supported nor validated by its manufacturer.

number of detector heads: 2

trans number of pixels: 1 # 1 in case of monolythic
trans pixel size: 540.0 # Given in mm
trans gap size: 0 # Given in mm

axial number of pixels: 1
axial pixel size: 400.0 # Given in mm
axial gap size: 0 # Given in mm

detector depth: 20

# Distance between the center of rotation (COR) of the scanner and the surface of a detection head in mm
scanner radius: 264.5,264.5 # Head 1 and 2 ROR for patient

# Collimator configuration

head1:
trans focal model: constant
trans number of coef model: 1
trans parameters: 0 # focal distance in mm, 0 for parallel
axial focal model: constant
axial number of coef model: 1
axial parameters: 0

head2:
trans focal model: constant
trans number of coef model: 1
trans parameters: 0 # focal distance in mm, 0 for parallel
axial focal model: constant
axial number of coef model: 1
axial parameters: 0

voxels number transaxial        : 128 # optional (default is the half of the scanner radius)
voxels number axial                : 128 # optional (default is length of the scanner computed from the given parameters)

field of view transaxial        : 613.7856 # optional (default is the half of the scanner radius)
field of view axial                : 613.7856 # optional (default is length of the scanner computed from the given parameters)
```

For the patient SPECT simulation, we set a translation of the `SPECThead` of 450 mm. The center of the SPECThead to center of the colimator is 158.5 mm and the collimator length is 54 mm. Therefore, the radius from COR to the front of collimator is 264.5 mm. 

Another argument `-t` can be provided to use only the true photons (i.e. unscattered), this will give a perfect scatter-corrected image to reconstruct. 

Running ```castor-GATErootToCastor``` executable  will generate a CASToR datafile (.Cdf) and header (.Cdh). The CASToR datafile can then be reconstructed with the `castor-recon` executable. 

The reconstruction can be run by proving `castor-recon` the following arguments: <br />

```
castor-recon
[Main options:]
-df castor_datafile.Cdh
-fout output_filename (or can use -dout to give output directory)
-it iterations:subsets e.g. 6:15
-dim dim_x,dim-y,dim_z (Number of voxels in each direction e.g. 128,128,128)
-vox voxel_size_x,voxel_size_y,voxel_size_z (Voxel size in each dimension, in mm)
[Optional extras:]
-opti MLEM (the optimiser to use, see -help-opti for all options)
-conv parameters;when (give image convolver parameters e.g. gaussian,7.,7.,5.::psf. See castor-recon -help-conv for all options. when states when the colvolver is applied, e.g. in psf) 
-atn linear_attenuation_coefficient.hdr (option to provide header file for linear attenuation image in units of /cm)
-th num_threads (set the number of threads for parallel computing, set to 0 for maximum available)
-proj incrementalSiddon (the projector to be used for forward and back projection Siddon is default, see castor-recon -help-proj for all options)
-vb verbosity (from 0 - no output to 5 - all events)
```
The gaussian convolution `gaussian,7.,7.,5.::psf` sets a classic stationary Gaussian kernel with transaxial FWHM 7 mm, axial FWHM 7 mm and 5 sigmas. 

`caator-recon` will write an image and headerfile for each iteration up to the total number specified. 
