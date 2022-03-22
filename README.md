# Diffusion MRI Acquisition Protocol and Data Analysis Pipeline

## Introduction
We describe a Magnetic Resonance Imaging (MRI) acquisition protocol and 
data analysis pipeline for diffusion tractography in the human brain.

## MRI Acquisition Protocol
Diffusion weighted MRI data are acquired on a 3T MRI system with gradient coils 
capable of producing an amplitude of 80 mT/m at a slew rate of 200 T/m/s using
a multi-band spin-echo echo-planar imaging (SE-EPI) pulse sequence.

Five separate series are acquired:
    
| Series | Phase encode direction | b-value | number of directions | b-value | number of directions |  
|---|---|---|---|---|---|
| 1 | anterior-posterior | 2500 | 75 | 0 | 5 |
| 2 | posterior-anterior | 0 | 4 | | |
| 3 | anterior-posterior | 1700 | 55 | 0 | 4 |
| 4 | anterior-posterior | 1000 | 30 | 0 | 3 |
| 5 | anterior-posterior | 400 | 18  | 0 | 2 |

Uniformly distributed diffusion gradient directions were created using 
[dirgen](https://mrtrix.readthedocs.io/en/latest/reference/commands/dirgen.html). 

The other acquisition parameters are kept the same for each sequence as described in more detail below:

1. Contrast:

    | Parameter | Value |
    |-----------|-------|
    | TE        | 82 ms |
    | TR        | 3500 ms |
    | Averages  | 1     |
    | Fat Saturation | On |

2. Resolution:

    | Parameter | Value |
    |-----------|-------|
    | Field-of-view (readout)| 220 mm |
    | Field-of-view (phase)  | 100 %  |
    | Slice thickness  | 2.5 mm |
    | Number of slices | 54 |
    | Matrix size | 88 by 88 |
    | Phase partial Fourier | 7/8 |
    | Interpolation | Off |
    | In plane acceleration factor | 2 |
    | Slice acceleration (multi-band) factor | 2 |

3. Sequence

    | Parameter | Value |
    |-----------|-------|
    | Echo spacing | 0.58 ms |
    | Bandwidth | 2030 Hz  |
    | RF pulse type | Normal |
    | Gradient mode | Performance |

## Data Analysis Pipeline
Diffusion weighted data are analysed using 
[MRtrix3](https://www.mrtrix.org/) and 
[FSL](https://fsl.fmrib.ox.ac.uk/fsl/) as described below:

1. Make a directory to store the analysis data, e.g. `mrtrix3-analysis`:
    ```bash
    mkdir mrtrix3-analysis
    ```

2. Change to the new directory:
    ```bash
    cd mrtrix3-analysis
    ```

3. Convert the [DICOM](https://dicom.nema.org/) data stored in `dcm_directory` to 
[mif](https://mrtrix.readthedocs.io/en/latest/getting_started/image_data.html) 
format using 
[mrconvert](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrconvert.html)
(see the [MRtrix3 documentation](https://mrtrix.readthedocs.io/en/latest/tips_and_tricks/dicom_handling.html)
for further information regarding DICOM to mif conversion):
    1. Convert the b = 0 s/mm^2 data acquired with phase-encode (PE) direction 
    posterior-anterior (PA) from DICOM to mif:
        ```bash
        mrconvert dcm_directory b0_PA.mif 
        ```
    2. Convert the remaining diffusion weighted images with phase-encode 
    direction anterior-posterior (AP) from DICOM to mif:
        ```bash
        mrconvert dcm_directory dwi_AP.mif 
        ```
       If you hae acquired multiple b-values as separate series you can select 
       them all in a list when prompted by 
       [mrconvert](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrconvert.html)

4. Concatenate the AP and PA volumes using 
[mrcat](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrcat.html):
    ```bash
    mrcat dwi_AP.mif  b0_PA.mif  dwi.mif
    ``` 
   
5. Denoise the data using 
[dwidenoise](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwidenoise.html):

    *As explained in the [MRtrix3 documentation](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwidenoise.html) 
    image denoising must be performed as the first step of the image-processing pipeline.* 
    
    1. Perform denoising:
        ```bash
        dwidenoise dwi.mif dwi_denoise.mif 
        ```
    2. Calculate the residual using 
    [mrcalc](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrcalc.html):
        ```bash
        mrcalc dwi.mif dwi_denoise.mif -subtract dwi_denoise_res.mif
        ```
    3. Visually check the denoised data and residual using 
    [mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
        ```bash
        mrview -mode 2 -load dwi.mif -interpolation 0 -load dwi_denoise.mif  -interpolation 0 -load dwi_denoise_res.mif  -interpolation 0
        ```
       
6. Remove the Gibbs ringing artefact the DWI data with 
[mrdegibbs](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrdegibbs.html):  
  
    *As explained in the [MRtrix3 documentation](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrdegibbs.html) 
    it is designed to run on data directly after it has been 
    reconstructed by the scanner, before any interpolation of any kind has 
    taken place i.e. you should run this command before any form of motion 
    correction (e.g. before 
    [dwifslpreproc](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwifslpreproc.html)). Similarly, if you intend 
    running [dwidenoise](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwidenoise.html), you should run this command afterwards, since it has 
    the potential to alter the noise structure, which would impact on 
    [dwidenoise's](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwidenoise.html) performance:*

    1. Remove Gibbs ringing artefact:
        ```bash
        mrdegibbs dwi_denoise.mif dwi_degibbs.mif
        ```
    2. Visually check the data using 
    [mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
        ```bash
        mrview -mode 2 -load dwi_denoise.mif  -interpolation 0 -load dwi_degibbs.mif  -interpolation 0
        ```

7. Correct for susceptibility and eddy current induced distortions and subject 
movements with [FSL](https://fsl.fmrib.ox.ac.uk/fsl/) tools
[topup](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/topup) and 
[eddy](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/eddy) via 
[dwifslpreproc](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwifslpreproc.html):
    
    1. Extract all the b=0 s/mm^2 AP and PA volumes following the removal of the 
    Gibbs ringing artefact with 
    [dwiextract](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwiextract.html):
        ```bash
        dwiextract -bzero dwi_degibbs.mif b0_degibbs.mif
       ```
    2. Determine the total number of b=0 s/mm^2 volumes (of either phase-encode 
    direction) using 
    [mrinfo](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrinfo.html) 
    (we'll refer to this as N_B0):
        ```bash
        mrinfo -shell_sizes b0_degibbs.mif
        ```
    3. Determine the number of b=0 s/mm^2 volumes with phase-encode direction
    PA using [mrinfo](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrinfo.html) 
    (we'll refer to this as N_B0_PA):
        ```bash
        mrinfo -shell_sizes b0_PA.mif
       ``` 
    4. Extract the last 2*N_B0_PA images e.g. 3 b=0 s/mm^2 volumes with 
    phase-encode direction AP followed by 3 b=0 s/mm^2 volumes with 
    phase-encode direction PA using 
    [mrconvert](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrconvert.html):
        ```bash
        mrconvert b0_degibbs.mif -coord 3 (N_B0-2*N_B0_PA):(N_B0-1) b0_degibbs_AP_PA.mif
        ```
    5. Visually check that the resulting data has N_B0 images with phase-encoding 
    AP followed by N_B0 images with phase-encoding PA using 
    [mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
        ```bash
        mrview -mode 2 -load b0_degibbs_AP_PA.mif -interpolation 0
        ```
    6. Determine the number of volumes with phase-encoding direction AP using
    [mrinfo](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrinfo.html)
     (we'll refer to this as N_AP):
        ```bash
        mrinfo -size dwi_AP.mif
        ```
    7. Extract the volumes with phase-encoding direction AP from the unringed 
    diffusion data using 
    [mrconvert](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrconvert.html):
        ```bash
        mrconvert dwi_degibbs_file.mif -coord 3 0:N_AP dwi_degibbs_AP.mif
        ```
    8. Determine phase-encode direction (PE_DIR) using
     [mrinfo](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrinfo.html) 
     e.g.`j-`:
        ```bash
        mrinfo dwi_AP.mif -all
        ```
    9. Run [dwifslpreproc](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwifslpreproc.html),
     using PE_DIR from above:
        ```bash
        dwifslpreproc dwi_degibbs_AP.mif dwi_tu_eddy.mif -rpe_pair -se_epi b0_degibbs_AP_PA.mif -pe_dir PE_DIR -eddy_options \"--slm=linear --data_is_shelled\"
        ```
    10. Visually check the resulting data using 
    [mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
        ```bash
        mrview -mode 2 -load dwi_degibbs.mif -interpolation 0 -load dwi_tu_eddy.mif -interpolation 0
        ``` 

8. Estimate the response functions of white matter (WM), grey matter (GM) and 
cerebro-spinal fluid (CSF) for spherical deconvolution using 
[dwi2response](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwi2response.html):

    *If you only have single-shell diffusion data then there are two different 
    options for data processing described on the MRtrix3 discussion boards 
    [a](https://community.mrtrix.org/t/lmaxes-specified-does-not-match-number-of-tissues/500/13) and 
    [b](https://community.mrtrix.org/t/wm-odf-and-response-function-with-dhollander-option---single-shell-versus-multi-shell/572/5).
    Option a uses the tournier algorithm for the WM response and csd for 
    single-shell single-tissue CSD. Whereas option b uses the dhollander algorithm
    for WM, GM and CSF responses and msmt_csd for multi-tissue CSD with only WM and 
    CSF. Option b is preferable because essentially it makes all false positive 
    "WM FODs" in the CSF disappear, and improves the quality of the WM FODs 
    when partial voluming with extra-cellular free water happens*

    1. Calculate the response using the dhollander algorithm:
        ```bash
        dwi2response dhollander dwi_tu_eddy.mif response_wm.txt response_gm.txt response_csf.txt -voxels response_voxels.mif
        ```

    2. Visually check the response functions data using 
    [shview](https://mrtrix.readthedocs.io/en/latest/reference/commands/shview.html):
        ```bash
        shview response_wm.txt
        shview response_gm.txt
        shview response_csf.txt
        ```
    3. Visually check the response voxels using 
    [mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
        ```bash
        mrview -mode 2 -load dwi_tu_eddy.mif -interpolation 0 -overlay.load response_voxels.mif -overlay.interpolation 0
        ```
9. Upsample the DWI data with 
[mrgrid](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrgrid.html):
    
    *As explained in the [MRtrix3 documentation](https://mrtrix.readthedocs.io/en/latest/fixel_based_analysis/mt_fibre_density_cross-section.html) upsampling DWI data before 
    computing FODs increases anatomical contrast and improves downstream 
    template building, registration, tractography and statistics. They 
    recommend upsampling to an isotropic voxel size of 1.3 mm for human brains.*
    ```bash
     mrgrid dwi_tu_eddy.mif regrid -voxel 1.3 dwi_tu_eddy_upsamp.mif
    ```
10. Creating a brain mask with FSL [bet](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/BET/UserGuide):

    *dwi2mask in combination with dwibiascorrect with -fsl option doesn't seem 
    to be very reliable so we use bet instead.*
    
    1. Extract the first b=0 s/mm^2 volume from the upsampled data in NIfTI 
    format using [dwiextract](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwiextract.html):
        ```bash
        dwiextract -bzero dwi_tu_eddy_upsamp.mif - | mrconvert -strides -1,2,3 - -coord 3 0 b0_upsamp.nii
        ```    
    2. Create a brain mask with bet with `f` and `g` parameters set to their default values:
        ```bash
        bet b0_upsamp.nii b0_upsamp_mask.nii.gz -n -m
        ```
    3. Visually check the resulting brain mask using 
    [fsleyes](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FSLeyes): 
        ```bash
        fsleyes b0_upsamp.nii b0_upsamp_mask.nii.gz -cm yellow -a 90
        ```
    4. Repeat steps ii and iii with different `f` and `g` parameters if the brain
    mask is not correct
    
    5. Convert the mask form NIfTI to mif format using 
    [mrconvert](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrconvert.html):
        ```bash
        mrconvert b0_upsamp_mask.nii.gz b0_upsamp_mask.mif
        ```
    6. Check the resulting mif format mask using 
    [mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
        ```bash
        mrview -mode 2 -load b0_upsamp.mif -interpolation 0 -overlay.load  -overlay.interpolation 0' b0_upsamp_mask.mif
        ```
  
11. Estimate the diffusion tensor with 
[dwi2tensor](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwi2tensor.html):
    ```bash
    dwi2tensor -mask b0_upsamp_mask.mif dwi_tu_eddy_upsamp.mif dt.mif
    ```
12. Calculating the fractional anisotropy (FA) and first eigenvector (EV) maps with 
[tensor2metric](https://mrtrix.readthedocs.io/en/latest/reference/commands/tensor2metric.html):
    ```bash
    tensor2metric -mask b0_upsamp_mask.mif -fa fa.mif -ev ev.mif dt.mif
    ```
13. Visually check the resulting FA and EV maps using 
[mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
    ```bash
    mrview -mode 2 -load fa.mif -interpolation 0 -load ev.mif -interpolation 0 -odf.load_tensor dt.mif
    ```
14. Perform Multi-Shell Multi-Tissue Constrained Spherical Deconvolution with 
[dwi2fod](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwi2fod.html):
    1. If the data is single-shell i.e. b-values of 0 and 2500 s/mm^2:  
        ```bash
        dwi2fod msmt_csd -mask b0_upsamp_mask.mif dwi_tu_eddy_upsamp.mif response_wm.txt wm_fod.mif response_csf.txt csf_fod.mif
        ```
    2. Or if the data is multi-shell i.e. b-values of 0, 700, 1400 and 2500 s/mm^2:
        ```bash
        dwi2fod msmt_csd -mask b0_upsamp_mask.mif dwi_tu_eddy_upsamp.mif response_wm.txt wm_fod.mif response_gm.txt gm_fod.mif response_csf.txt csf_fod.mif
        ```
    3. Visually check the resulting WM FODs using 
    [mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
        ```bash   
        mrview -mode 2 -load fa.mif -interpolation 0 -load ev.mif -interpolation 0 -odf.load_sh wm_fod.mif
        ```
       
15. Produce a map of the tissue volume fractions (only if data is multi-shell)
using [mrconvert](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrconvert.html) and 
[mrcat](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrcat.html):
    1. Create the map of tissue volume fractions:
        ```bash
        mrconvert -coord 3 0 wm_fod.mif - | mrcat csf_fod.mif gm_fod.mif - tissue_vf.mif
        ```
    2. Visually check the result using 
    [mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
        ```bash
        mrview -mode 2 -load tissue_vf.mif -interpolation 0
        ```
       
16. Draw regions-of-interest (ROIs) on the FA and EV maps using
[mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
    1. Open [mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
        ```bash   
        mrview -mode 2 -load fa.mif -interpolation 0 -load ev.mif -interpolation 0 -odf.load_sh wm_fod.mif
        ```
    2. From the Tool menu select ROI editor
    3. Create a new ROI and edit it (e.g. an ROI in the right-hand motor area in the left hemisphere)
    4. Save the ROI in mif format (e.g. `RHandM1.mif`)
    5. Create another new ROI and edit it (e.g. an ROI in the cerebral peduncles)    
    6. Save the ROI in mif format (e.g. `L-Ped.mif`)
    
17. Perform probabilistic tractography using
[tckgen](https://mrtrix.readthedocs.io/en/latest/reference/commands/tckgen.html) 
e.g. for the ROIs created above for the corticospinal tract related to movement 
of the right hand:
    ```bash   
    tckgen wm_fod.mif -seed_image RHandM1.mif -include L-Ped.mif mrtrix3-RHandM1ToPed.tck 
    ```

    Commonly used options include `-cutoff`, `-seed_unidirectional` and `-stop`    

18. View the tract streamlines using 
[mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
    ```bash   
    mrview -mode 2 -load fa.mif -interpolation 0 -tractography.load mrtrix3-RHandM1ToPed.tck
    ```

The streamlines can also be converted to images (mif or NIfTI format) as described below:

1. Convert the streamlines (tck) to an image (mif) using 
[tckmap](https://mrtrix.readthedocs.io/en/latest/reference/commands/tckmap.html):
    ```bash   
    tckmap -template fa.mif mrtrix3-RHandM1ToPed.tck mrtrix3-RHandM1ToPed.mif
    ```
   
2. Normalise the voxel intensities based on the track count 
(to help with applying a consistent threshold when viewing results):
    1. Determining the number of tracts using 
    [tckinfo](https://mrtrix.readthedocs.io/en/latest/reference/commands/tckinfo.html) (we'll refer to this as N_tracts):
        ```bash   
        tckinfo -count mrtrix3-RHandM1ToPed.tck
        ```
    2. Normalise using [mrcalc](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrcalc.html):
        ```bash   
        mrcalc mrtrix3-RHandM1ToPed.mif N_tracts -div mrtrix3-RHandM1ToPed_norm.mif
        ```
       
3. View the tract image using 
[mrview](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrview.html):
    ```bash   
    mrview -mode 2 -load fa.mif -interpolation 0 -overlay.load mrtrix3-RHandM1ToPed_norm.mif
    ```     
   
4. Convert the normalised mif to NIfTI format using 
[mrconvert](https://mrtrix.readthedocs.io/en/latest/reference/commands/mrconvert.html):
    ```bash   
    mrconvert -strides -1,2,3 mrtrix3-RHandM1ToPed_norm.mif mrtrix3-RHandM1ToPed_norm.nii
    ```
       
## Authors and Acknowledgements
[Dr Stephen Wastling](mailto:stephen.wastling@nhs.net) and 
[Dr Laura Mancini](mailto:laura.mancini@nhs.net) with many thanks
to the developers of [FSL](https://fsl.fmrib.ox.ac.uk/fsl/) and 
[MRtrix3](https://www.mrtrix.org/).