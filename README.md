# Reconstruction-Of-Nuclear-Medicine-Imaging-Systems-Simulated-Acquisition-Scintigraphy-SPECT

Steps to exaplain:

We will explain in this tutorial how to reconstruct some of our GATE simulation [benchmarks]{https://github.com/BenAuer2021/Simulation-Of-Nuclear-Medicine-Imaging-Systems-Scintigraphy-SPECT} with [CASTOR]{https://castor-project.org/}.

# 1. Convert the GATE ROOT Output into CASTOR Histogram File 

## Adapt the CASTOR Toolkit to be used with our simulated data
CASTOR provides a tool to create a CASTOR histogram datafile directly from a GATE macro and root file: `castor-GATErootToCastor`. However, by default this code utilizes the Singles TTree. In our [simulation]{https://github.com/BenAuer2021/Simulation-Of-Nuclear-Medicine-Imaging-Systems-Scintigraphy-SPECT} we did set an energy range for the Photopeak TTree only, the Singles TTree was based on the full-spectrum. This approach is convenient, as it does not require to re-run the simulation if the energy window needs to be changed. The energy window can be applied while processing the ROOT file, for more information please see this [page]{https://github.com/BenAuer2021/Tools-To-Analyze-Simulation-Output}.

We thus adapted the `castor-GATErootToCastor` code to use the 'Photopeak TTree' in place of the 'Singles TTree'. This was done by modifying line 1386 by
```ruby
GEvents[iFic] = (TTree*)Tfile_root[iFic]->Get("Photopeak"); // Name of the Tree related to the photopeak
```
When then modified the 'CMakeLists.txt' and recompiled CASTOR to create another executable for the photopeak 'castor-GATERootToCastor-Photopeak'.

## 2. Create the CASTOR Scanner and Histogram SPECT Files

To run the 'castor-GATErootToCastor-Photopeak', the following options need to be set,

```castor-GATErootToCastor-Photopeak -i path/to/ifile.root -o path/to/outfile -m path/to/macrofile.mac -geo -s scanner_alias -sp_bins bins_x,bins_y``` <br />
where <br />
`path/to/ifile.root` is the root output file from Gate. <br />
`path/to/outfile` is the base name to save the CASToR datafile to. <br />
`path/to/macrofile.mac` is the Gate macro file used to generate the root file. <br />
`scanner_alias` corresponds to a `scanner_alias.geom` file in your `castor/config/scanner/` directory. If the file does not exist '-geo' option will create it.  <br />
`geo` will generate a CASToR geometry file from the provided GATE macro file(s). If the scanner_alias.geom file exist in the `castor/config/scanner/` directory, this option can be removed.  <br />
`bins_x,bins_y` are the transaxial and axial number of bins for projections, separated by a comma.

If the '-geo' option is specified, CASTOR will create a CASTOR scanner geometry file in the 'castor/config/scanner/' from the GATE '.mac' file. Note that in order to determine the distance of the collimator to the center of rotation, CASToR expects the SPECTHead dimension to be specified in cm. However, this distance is overestimated as it is based on the SPECTHead shift and does not take into account the SPECTHead transaxial dimension. We strongly recomend to edit the line 'scanner radius: 264.5,264.5' of the 'scanner_alias.geom' file generating, by substracting half of the transaxial dimension of the SPECTHead. For example, for the following definition in GATE,
```ruby
/gate/world/daughters/name SPECThead
/gate/world/daughters/insert box
/gate/SPECThead/geometry/setXLength 37.5 cm
/gate/SPECThead/geometry/setYLength 64 cm
/gate/SPECThead/geometry/setZLength 53 cm
/gate/SPECThead/placement/setTranslation -32.25 0  0 cm
/gate/SPECThead/setMaterial Air
/gate/SPECThead/vis/setColor cyan
```
In this example, we set in our simulation macro a translation of the `SPECThead` of 322.5 mm. The center of the SPECThead to center of rotation is 187.5 mm and the half dimension of the SPECTHead is 375 mm along the transaxial axis. Therefore, the distance of the front surface of the collimator to the center of rotation  is 135 mm. CASTOR will estimate the 'scanner radius' to be 322.5 mm, however this distance should be equal to 135 mm (322.5-37.5/2).

Comments after commands can also cause issues so make sure to remove all comments from the GATE macro (*.mac) file.

The `scanner_alias.geom` file defines the components of the system. Below is an example for the BrightView system used for all our simulation described [here]{https://github.com/BenAuer2021/Simulation-Of-Nuclear-Medicine-Imaging-Systems-Scintigraphy-SPECT/edit/main/README.md} 
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
scanner radius: 135.0, 135.0 # Head 1 and 2 ROR is 13.5 cm for brain application

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

### OPTIONAL
#voxels number transaxial        : 128 # optional (default is the half of the scanner radius)
#voxels number axial                : 128 # optional (default is length of the scanner computed from the given parameters)
#field of view transaxial        : 613.7856 # optional (default is the half of the scanner radius)
#field of view axial                : 613.7856 # optional (default is length of the scanner computed from the given parameters)
```
As it can be seen the collimator specification (Resolution Recovery) are not modeled currently in CASTOR. The same CASTOR '.geom' can thus be used with the BrightView system equiped with multiple parrallel-hole collimators we created and available [here]{https://github.com/BenAuer2021/Simulation-Of-Nuclear-Medicine-Imaging-Systems-Scintigraphy-SPECT/edit/main/README.md} to the extent that the radius of rotation (ROR) remain similar.

For brain perfusion and DaT we use a ROR of 13.5 cm. However, for our quality control phantom simulation (Defrise, Jaszczak, Fillable Jaszczak, Derenzo) we used 22 cm ROR, the above line needs to be modified by 'scanner radius: 220.0, 220.0'. For our glioblastoma imaging benchmark we used 15.5 cm ROR, the above line  needs to be modified by 'scanner radius: 155.0, 155.0'. For our Bone imaging benchmark we used 41.25 cm ROR, the above line needs to be modified by 'scanner radius: 412.5, 412.5'. For our Lu-177 DOTATATE imaging benchmark we used 26.25 cm ROR, the above line needs to be modified by 'scanner radius: 262.5, 262.5'. We provide multiple '*.geom' files to be reconstructed for each of these scenario.

Another argument `-t` or '-ot' can be provided to use only the primary photons (i.e. unscattered) based on the simulation, this gives a perfect scatter-corrected image to reconstruct and can be seen as a perfect scatter rejection technique. However, reducing the number of counts will increase the overall noise in the reconstructed image. As we will see below, additional filtering can compensate for such effect. 

Running ```castor-GATErootToCastor-Photopeak``` executable  will generate a CASToR datafile (.Cdf) and header (.Cdh). For example, 
```ruby
home/benjamin/Documents/Software/CASTOR/castor_v3.1.1-build/castor-GATERootToCastor-Photopeak -i ../ROOT_Files/BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total.root -s SPECT_BRIGHTVIEW -o BRAIN_PERFUSION_CASTOR_230x170 -sp_bins 230,170 -m BRAIN_PERFUSION/Brightview_Main_LEHR_Tc99m_BrainPerfusion.mac  -vb 2
```
will generate the following output and the Histogram data BRAIN_PERFUSION_CASTOR_230x170.Cdh and BRAIN_PERFUSION_CASTOR_230x170.Cdf as it can be seen below,
<img width="811" alt="Screen Shot 2023-06-22 at 2 30 11 PM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/595c43e2-46d8-4797-a0a3-4916352779ea">

The file size of the binary histogram file 'BRAIN_PERFUSION_CASTOR_230x170.Cdf' is 72 MB. BRAIN_PERFUSION_CASTOR_230x170.Cdh is the header file and consist of the following information,
```ruby
Data filename: BRAIN_PERFUSION_CASTOR_230x170_df.Cdf
Number of events: 4692000
Data mode: histogram
Data type: SPECT
Start time (s): 0
Duration (s): 60
Scanner name: SPECT_BRIGHTVIEW
Number of bins: 230, 170
Number of projections: 120
Projection angles: 0, 3, 6, 9, 12, 15, 18, 21, 24, 27, 30, 33, 36, 39, 42, 45, 48, 51, 54, 57, 60, 63, 66, 69, 72, 75, 78, 81, 84, 87, 90, 93, 96, 99, 102, 105, 108, 111, 114, 117, 120, 123, 126, 129, 132, 135, 138, 141, 144, 147, 150, 153, 156, 159, 162, 165, 168, 171, 174, 177, 180, 183, 186, 189, 192, 195, 198, 201, 204, 207, 210, 213, 216, 219, 222, 225, 228, 231, 234, 237, 240, 243, 246, 249, 252, 255, 258, 261, 264, 267, 270, 273, 276, 279, 282, 285, 288, 291, 294, 297, 300, 303, 306, 309, 312, 315, 318, 321, 324, 327, 330, 333, 336, 339, 342, 345, 348, 351, 354, 357
Global distance camera surface to COR: 322.5
Calibration factor: 1
Isotope: unknown
Normalization correction flag: 0
Scatter correction flag: 0
Head rotation direction: CW
```
## 3. Reconstruction in CASToR 

Once the the Histogram CASTOR data are generated, we can reconstruct them via the `castor-recon` executable via the following command: <br />
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
The gaussian convolution `gaussian,7.,7.,5.::psf` sets a inter-recon stationary Gaussian kernel with a transaxial FWHM of 7 mm, axial FWHM of 7 mm and a kernel of 5 by 5 sigmas. `castor-recon` will write an image (*.img) and header file (*.hdr) for each iteration and subsets up to the total number specified (here '6:15', 6 iterations and 15 subsets. If the option '-oit -1' is specified only the last iteration image will be saved.

For example, if we have CASTOR Histogram file named 'BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_df.Cdh' generated for a brain perfusion phantom, the following command will reconstruct the data with 6 iterations 15 subests in a 128x128x132 format with a voxel size of 2.34x2.34x3.125 mm<sup>2</sup>. THis allows to cover the axial (400 mm) and transaxial (270 mm) field of view of the BrightView system for brain imaging. A Gaussian inter-recon filter of transaxial FWHM of 7 mm, axial FWHM of 7 mm and a kernel of 5 by 5 sigmas is applied. The option 'th 0' uses the maximum number of thread available on the computer.
```ruby
castor-recon -df BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_df.Cdh -fout OUTPUT -it 6:15 -dim 128,128,132 -vox 2.34,2.34,3.125 -conv gaussian,7.,7.,5.::psf -th 0 -vb 2 -opti MLEM -proj incrementalSiddon
```
CASTOR will produce a set of reconstructed images for each iterations. These reconstructed images can be imported in ImageJ and Amide via the following parameters,

<img width="513" alt="Screen Shot 2023-06-22 at 3 31 16 PM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/84b5ad85-6d56-433d-a8d3-03b14c671e32">

The '-atn' command can be added to perform attenuation correction during reconstruction. The attenuation map must be formated for CASTOR
```ruby
castor-recon -df BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_df.Cdh -fout OUTPUT_AC -it 6:15 -dim 128,128,132 -vox 2.34,2.34,3.125 -conv gaussian,7.,7.,5.::psf -th 0 -vb 2 -atn AllTissues_120x120x120_ForGate1_72x90x77_LinAttenCoeffsCm_140keV_float32_Livermore_rotate90.hdr -opti MLEM -proj incrementalSiddon
```
It is important to make sure the CASTOR reconstructed image and attenuation map are properly registered and similarly oriented.

<img width="510" alt="Screen Shot 2023-06-22 at 3 34 13 PM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/58a811b1-f526-400e-bda6-ac570f35a522">

To produce quantitative images, scatter correction (i.e. scatter rejection) can be added in addition to attenuation correction by using the CASTOR Histogram consisting solely of the primary photons, named 'BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_tSC_df.Cdh' below. The command line will be the following,
castor-recon -df BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_tSC_df.Cdh -fout BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_128,128,132_2.34,2.34,3.125_gaussian,7.,7.,5.::psf_AC_SC -it 6:15 -dim 128,128,132 -vox 2.34,2.34,3.125 -conv gaussian,7.,7.,5.::psf -th 0 -vb 2 -atn AllTissues_120x120x120_ForGate1_72x90x77_LinAttenCoeffsCm_140keV_float32_Livermore_rotate90.hdr -opti MLEM -proj incrementalSiddon

<img width="457" alt="Screen Shot 2023-06-22 at 3 41 25 PM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/e3d8e106-ed9c-470e-8674-bbcae16e27c1">

