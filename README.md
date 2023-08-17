# Reconstruction-Of-Nuclear-Medicine-Imaging-Systems-Simulated-Acquisition-Scintigraphy-SPECT

This tutorial is developed and maintained by Auer Benjamin from the Brigham and Women's Hospital and Harvard Medical School, Boston, MA, USA, and Pells Sophia from the University of Massachussets Chan Medical School, Worcester, MA, USA.

**Contact:** Auer, Benjamin Ph.D <bauer@bwh.harvard.edu>

Table of contents:
```diff
- 1. Convert the GATE ROOT Output into CASTOR Histogram File
-- 1.1. Adapt the CASTOR Toolkit to be used with our simulated data
-- 1.2. Create the CASTOR Scanner and Histogram SPECT Files
- 2. Reconstruction in CASToR
-- 2.1 Reconstruction without attenuation and scatter correction
-- 2.2 Reconstruction with attenuation correction
-- 2.3 Reconstruction with attenuation and scatter correction
- 3. Benchmarks
-- 3.1 Bone Imaging
-- 3.2 Fillable Jaszczak Phantom Imaging
-- 3.3 Brain Perfusion Imaging
-- 3.4 Brain DaT Imaging
-- 3.5 Brain Glioblastoma Imaging
-- 3.6 Dotatate Imaging
```
-----
We will explain in this tutorial how to reconstruct some of our GATE simulation [benchmarks](https://github.com/BenAuer2021/Simulation-Of-Nuclear-Medicine-Imaging-Systems-Scintigraphy-SPECT) with [CASToR](https://castor-project.org/).

# 1. Convert the GATE ROOT Output into CASToR Histogram File 

## 1.1. Adapt the CASToR Toolkit to be used with our simulated data
CASToR provides a tool to create a CASToR histogram datafile directly from a GATE macro and root file: `castor-GATErootToCastor`. However, by default this code utilizes the Singles TTree. In our [simulation](https://github.com/BenAuer2021/Simulation-Of-Nuclear-Medicine-Imaging-Systems-Scintigraphy-SPECT) we did set an energy range for the Photopeak TTree only, the Singles TTree was based on the full-spectrum. This approach is convenient, as it does not require to re-run the simulation if the energy window needs to be changed. The energy window can be applied while processing the ROOT file, for more information please see this [page](https://github.com/BenAuer2021/Tools-To-Analyze-Simulation-Output).

We thus adapted the `castor-GATErootToCastor` code to use the `Photopeak TTree` in place of the `Singles TTree`. This was done by modifying line 1386 by
```ruby
GEvents[iFic] = (TTree*)Tfile_root[iFic]->Get("Photopeak"); // Name of the Tree related to the photopeak
```
When then modified the `CMakeLists.txt` and recompiled CASToR to create another executable for the photopeak `castor-GATERootToCastor-Photopeak`.

## 1.2. Create the CASToR Scanner and Histogram SPECT Files

To run the `castor-GATErootToCastor-Photopeak`, the following options need to be set,

```castor-GATErootToCastor-Photopeak -i path/to/ifile.root -o path/to/outfile -m path/to/macrofile.mac -geo -s scanner_alias -sp_bins bins_x,bins_y``` <br />
where <br />
`path/to/ifile.root` is the root output file from Gate. <br />
`path/to/outfile` is the base name to save the CASToR datafile to. <br />
`path/to/macrofile.mac` is the Gate macro file used to generate the root file. <br />
`scanner_alias` corresponds to a `scanner_alias.geom` file in your `castor/config/scanner/` directory. If the file does not exist `-geo` option will create it.  <br />
`geo` will generate a CASToR geometry file from the provided GATE macro file(s). If the scanner_alias.geom file exist in the `castor/config/scanner/` directory, this option can be removed.  <br />
`bins_x,bins_y` are the transaxial and axial number of bins for projections, separated by a comma.

If the `-geo` option is specified, CASToR will create a CASToR scanner geometry file in the `castor/config/scanner/` from the GATE `.mac` file. Note that in order to determine the distance of the collimator to the center of rotation, CASToR expects the SPECTHead dimension to be specified in cm. However, this distance is overestimated as it is based on the SPECTHead shift and does not take into account the SPECTHead transaxial dimension. We strongly recomend to edit the line `scanner radius: 264.5,264.5` of the `scanner_alias.geom` file generating, by substracting half of the transaxial dimension of the SPECTHead. For example, for the following definition in GATE,
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
In this example, we set in our simulation macro a translation of the `SPECThead` of 322.5 mm. The center of the SPECThead to center of rotation is 187.5 mm and the half dimension of the SPECTHead is 375 mm along the transaxial axis. Therefore, the distance of the front surface of the collimator to the center of rotation  is 135 mm. CASToR will estimate the `scanner radius` to be 322.5 mm, however this distance should be equal to 135 mm (322.5-37.5/2).

Comments after commands can also cause issues so make sure to remove all comments from the GATE macro (*.mac) file.

The `scanner_alias.geom` file defines the components of the system. Below is an example for the BrightView system used for all our simulation described [here](https://github.com/BenAuer2021/Simulation-Of-Nuclear-Medicine-Imaging-Systems-Scintigraphy-SPECT/edit/main/README.md) 
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
As it can be seen the collimator specification (Resolution Recovery) are not modeled currently in CASToR. The same CASToR `.geom` can thus be used with the BrightView system equiped with multiple parrallel-hole collimators we created and available [here](https://github.com/BenAuer2021/Simulation-Of-Nuclear-Medicine-Imaging-Systems-Scintigraphy-SPECT/edit/main/README.md) to the extent that the radius of rotation (ROR) remain similar.

For brain perfusion and DaT we use a ROR of 13.5 cm. However, for our quality control phantom simulation (Defrise, Jaszczak, Fillable Jaszczak, Derenzo) we used 22 cm ROR, the above line needs to be modified by `scanner radius: 220.0, 220.0`. For our glioblastoma imaging benchmark we used 15.5 cm ROR, the above line  needs to be modified by `scanner radius: 155.0, 155.0`. For our Bone imaging benchmark we used 41.25 cm ROR, the above line needs to be modified by `scanner radius: 412.5, 412.5`. For our Lu-177 DOTATATE imaging benchmark we used 26.25 cm ROR, the above line needs to be modified by `scanner radius: 262.5, 262.5`. We provide multiple `*.geom` files to be reconstructed for each of these scenario.

Another argument `-t` or `-ot` can be provided to use only the primary photons (i.e. unscattered) based on the simulation, this gives a perfect scatter-corrected image to reconstruct and can be seen as a perfect scatter rejection technique. However, reducing the number of counts will increase the overall noise in the reconstructed image. As we will see below, additional filtering can compensate for such effect. 

Running ```castor-GATErootToCastor-Photopeak``` executable  will generate a CASToR datafile (.Cdf) and header (.Cdh). For example, 
```ruby
home/benjamin/Documents/Software/CASToR/castor_v3.1.1-build/castor-GATERootToCastor-Photopeak -i ../ROOT_Files/BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total.root -s SPECT_BRIGHTVIEW -o BRAIN_PERFUSION_CASToR_230x170 -sp_bins 230,170 -m BRAIN_PERFUSION/Brightview_Main_LEHR_Tc99m_BrainPerfusion.mac  -vb 2
```
will generate the following output and the Histogram data BRAIN_PERFUSION_CASToR_230x170.Cdh and BRAIN_PERFUSION_CASToR_230x170.Cdf.

https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/7bcba518-46fc-42ff-91e4-3a83e98ee27b

The file size of the binary histogram file `BRAIN_PERFUSION_CASToR_230x170.Cdf` is 72 MB. BRAIN_PERFUSION_CASToR_230x170.Cdh is the header file and consist of the following information,
```ruby
Data filename: BRAIN_PERFUSION_CASToR_230x170_df.Cdf
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
# 2. Reconstruction in CASToR 

Once the the Histogram CASToR data are generated, we can reconstruct them via the `castor-recon` executable via the following command: <br />
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
The gaussian convolution `gaussian,7.,7.,5.::psf` sets a inter-recon stationary Gaussian kernel with a transaxial FWHM of 7 mm, axial FWHM of 7 mm and a kernel of 5 by 5 sigmas. `castor-recon` will write an image (*.img) and header file (*.hdr) for each iteration and subsets up to the total number specified (here `6:15`, 6 iterations and 15 subsets. If the option `-oit -1` is specified only the last iteration image will be saved.

## 2.1 Reconstruction without attenuation and scatter correction
For example, if we have CASToR Histogram file named `BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_df.Cdh` generated for a brain perfusion phantom, the following command will reconstruct the data with 6 iterations 15 subests in a 128x128x132 format with a voxel size of 2.34x2.34x3.125 mm<sup>3</sup>. THis allows to cover the axial (400 mm) and transaxial (270 mm) field of view of the BrightView system for brain imaging. A Gaussian inter-recon filter of transaxial FWHM of 7 mm, axial FWHM of 7 mm and a kernel of 5 by 5 sigmas is applied. The option `th 0` uses the maximum number of thread available on the computer.
```ruby
castor-recon -df BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_df.Cdh -fout OUTPUT -it 6:15 -dim 128,128,132 -vox 2.34,2.34,3.125 -conv gaussian,7.,7.,5.::psf -th 0 -vb 2 -opti MLEM -proj incrementalSiddon
```
The following output will be generated for 2 OSEM iterations with 5 subsets,

https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/333ed05d-3916-414a-85db-441ccbb519a0

With the `6:15` option, CASToR will produce a set of reconstructed images for each iterations. These reconstructed images can be imported in ImageJ and Amide via the following parameters,

<img width="680" alt="Screen Shot 2023-08-17 at 8 56 48 AM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/64c263ce-095a-4f58-a71e-882bccf9a42d">


## 2.2 Reconstruction with attenuation correction
The `-atn` command can be added to perform attenuation correction during reconstruction. The attenuation map must be formated for CASToR
```ruby
castor-recon -df BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_df.Cdh -fout OUTPUT_AC -it 6:15 -dim 128,128,132 -vox 2.34,2.34,3.125 -conv gaussian,7.,7.,5.::psf -th 0 -vb 2 -atn AllTissues_120x120x120_ForGate1_72x90x77_LinAttenCoeffsCm_140keV_float32_Livermore_rotate90.hdr -opti MLEM -proj incrementalSiddon
```
It is important to make sure the CASToR reconstructed image and attenuation map are properly registered and similarly oriented.

## 2.3 Reconstruction with attenuation and scatter correction
To produce quantitative images, scatter correction (i.e. scatter rejection) can be added in addition to attenuation correction by using the CASToR Histogram consisting solely of the primary photons, named `BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_tSC_df.Cdh` below. The command line will be the following,
```ruby
castor-recon -df BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_tSC_df.Cdh -fout BV_LEHR_ADAC_HofBrain_14.75BqScale_fluo0Nested_1sRuns180deg_mono140keV-total_SPECT_BRIGHTVIEW_230x170_128,128,132_2.34,2.34,3.125_gaussian,7.,7.,5.::psf_AC_SC -it 6:15 -dim 128,128,132 -vox 2.34,2.34,3.125 -conv gaussian,7.,7.,5.::psf -th 0 -vb 2 -atn AllTissues_120x120x120_ForGate1_72x90x77_LinAttenCoeffsCm_140keV_float32_Livermore_rotate90.hdr -opti MLEM -proj incrementalSiddon
```

https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/a6776f5e-92f5-426b-bdd1-4efd7b02812e

A comparison of the different degree of correction is provided below,

<img width="644" alt="Screen Shot 2023-08-17 at 9 09 41 AM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/31c1c510-efb4-4161-9f3a-5841e7e057d7">

# 3. Benchmarks
We provide multiple example of reconstruction in CASToR of GATE simulated data available [here](https://github.com/BenAuer2021/Simulation-Of-Nuclear-Medicine-Imaging-Systems-Scintigraphy-SPECT).

## 3.1 Bone Imaging
In this example, we simulated in GATE a whole-body Tc-99m MDP Bone scan with 3 bed positions (400 mm axial length each) with the BrightView system equipped with LEHR collimator. Each of the 3 bed position acquisition consisted of 64 views over 360 degree and the radius of rotation was 41.25 cm. The first bed position ( `pos1` centered on the leg region) scan recorded 6,705,068 Cts, the second bed position ( `pos0` centered on the torso and neck region) included 11,881,614 Cts, and the third bed position (`pos 2` centered on the abdominal region) consisted of 13,030,131 Cts. Attenuation in the phantom was not modeled in the simulation in order to improve computation efficiency. Simulated projections were reconstructed in 360x360x132 voxels of 2.34x2.34x3.125 mm<sup>3</sup> with a Gaussian intra-recon filter of 7 mm by 7 mm and a kernel size of 5 by 5 and 2 iterations/15 subsets. We provide the Histogram CASToR files (*.Cdf, *.Cdh) and the scanner *.geom file (**Bone.zip**).

<img width="665" alt="Screen Shot 2023-08-17 at 9 11 27 AM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/5125f2fb-d199-49e4-b730-cef154f1ca28">

## 3.2 Fillable Jaszczak Phantom Imaging
In this example, we simulated in GATE with the BrightView system equipped with LEHR collimator an Tc-99m acquisition with the fillable Jaszczak phantom with and without background activity described [here](https://github.com/BenAuer2021/Phantoms-For-Nuclear-Medicine-Imaging-Simulation). The acquisition consisted of 120 views over 360 degree and the radius of rotation was 22 cm. A total of 46,632.127 Cts and 15,443,078 Cts were recorded with and without background activity, respectively. Attenuation in the phantom was not modeled in the simulation in order to improve computation efficiency. Simulated projections were reconstructed in 128x128x132 voxels of 2.34x2.34x3.125 mm<sup>3</sup> with a Gaussian intra-recon filter of 5 mm by 5 mm and a kernel size of 5 by 5 and 2 iterations/15 subsets. We provide the Histogram CASToR files (*.Cdf, *.Cdh) and the scanner *.geom file (**Fillable_ Jaszczak_wBackgroundActivity.zip** && **Fillable_ Jaszczak.zip**).

<img width="588" alt="Screen Shot 2023-08-17 at 9 13 36 AM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/4d07781a-4650-485d-ab56-ea29996f78c7">

## 3.3 Brain Perfusion Imaging
In this example, we simulated in GATE with the BrightView system equipped with LEHR-VXHR collimator an Tc-99m HMPAO brain perfusion acquisition with the brain perfusion phantom described [here](https://github.com/BenAuer2021/Phantoms-For-Nuclear-Medicine-Imaging-Simulation). The acquisition consisted of 120 views over 360 degree and radius of rotation was 13.5 cm. A total of 5,250,896 Cts was recorded. Attenuation in the phantom was modeled in the simulation. The simulated projections were reconstructed in 128x128x132 voxels of 2.34x2.34x3.125 mm<sup>3</sup> with a Gaussian intra-recon filter of 5 mm by 5 mm and a kernel size of 5 by 5 and 2 iterations/15 subsets. We provide the Histogram CASToR files with and without scatter (*.Cdf, *.Cdh), the attenuation map (*.hdr, *.img) and the scanner *.geom file (**Brain_Perfusion.zip**).

<img width="644" alt="Screen Shot 2023-08-17 at 9 14 08 AM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/e14f69af-8485-490e-b842-1ecd61bec36c">

<img width="749" alt="image" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/70d803bf-9798-49f2-aa1c-f03e82edc723">

## 3.4 Brain DaT Imaging
In this example, we simulated in GATE with the BrightView system equipped with LEHR and MEGP collimators an I-123 Ioflupane brain DaT acquisition with the DaT phantom described [here](https://github.com/BenAuer2021/Phantoms-For-Nuclear-Medicine-Imaging-Simulation). The acquisition consisted of 120 views over 360 degree and radius of rotation was 13.5 cm. A total of 6,640,150 Cts and 1,601,801 Cts were recorded for acquisition with MEGP and LEHR collimators. Attenuation in the phantom was not modeled in the simulation in order to improve computation efficiency. The simulated projections were reconstructed in 128x128x132 voxels of 2.34x2.34x3.125 mm<sup>3</sup> with a Gaussian intra-recon filter of 7 mm by 7 mm and a kernel size of 5 by 5 and 2 iterations/15 subsets. We provide the Histogram CASToR files (*.Cdf, *.Cdh) and the scanner *.geom file (**DaT.zip**).

<img width="639" alt="Screen Shot 2023-08-17 at 9 15 27 AM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/b3075aa0-0ea3-40b5-b77e-07a91ab8cc42">

## 3.5 Brain Glioblastoma Imaging
In this example, we simulated in GATE with the BrightView system equipped with HEGP collimator an I-131 brain glioblastoma acquisition with the glioblastoma phantom described [here](https://github.com/BenAuer2021/Phantoms-For-Nuclear-Medicine-Imaging-Simulation). The acquisition consisted of 120 views over 360 degree and radius of rotation was 15.5 cm. A total of 11,122,811 Cts was recorded. Attenuation in the phantom was not modeled in the simulation in order to improve computation efficiency. The simulated projections were reconstructed in 128x128x132 voxels of 2.34x2.34x3.125 mm<sup>3</sup> with a Gaussian intra-recon filter of 7 mm by 7 mm and a kernel size of 5 by 5 and 1-4 iterations with 15 subsets. We provide the Histogram CASToR files (*.Cdf, *.Cdh) and the scanner *.geom file (**Glioblastoma.zip**).

<img width="576" alt="Screen Shot 2023-08-17 at 9 16 52 AM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/260a456b-a6b5-40f5-90ff-97cb550ce218">

## 3.6 Dotatate Imaging
In this example, we simulated in GATE with the BrightView system equipped with MEGP collimator an Lu-177 Dotatate acquisition with the patient data phantom described [here](https://github.com/BenAuer2021/Phantoms-For-Nuclear-Medicine-Imaging-Simulation). The acquisition consisted of 64 views over 360 degree and radius of rotation was 26.25 cm. A total of 1,617,529 Cts was recorded. Attenuation in the phantom was modeled in the simulation. The simulated projections were reconstructed in 280x280x132 voxels of 2.34x2.34x3.125 mm<sup>3</sup> with a Gaussian intra-recon filter of 7 mm by 7 mm and a kernel size of 5 by 5. We provide the Histogram CASToR files with and without scatter (*.Cdf, *.Cdh), the attenuation map (*.hdr, *.img), and the scanner *.geom file (**DOTATATE.zip**).

<img width="434" alt="Screen Shot 2023-08-17 at 9 17 21 AM" src="https://github.com/BenAuer2021/Reconstruction-Of-Simulated-Nuclear-Medicine-Acquisition-Scintigraphy-SPECT/assets/84809217/a05203a9-0ab4-4153-82de-caaa7e540234">



