# Rat rsfMRI Preprocessing Toolbox
Preprocessing codes for the rat rs-fMRI database (www.nitrc.org/projects/rat_rsfmri). 

## Quick Start
This section is a quick-start guide to this toolbox. More details can be found in code comments.
### Step 1: Download prerequisites:
Download the following software packages before using the toolbox:
- GIFT ICA toolbox. (https://www.nitrc.org/projects/gift) 
- jsonlab. (https://www.mathworks.com/matlabcentral/fileexchange/33381-jsonlab-a-toolbox-to-encode-decode-json-files)
- jq. (https://stedolan.github.io/jq/)
- Tools for NIfTI and ANALYZE image (https://www.mathworks.com/matlabcentral/fileexchange/8797-tools-for-nifti-and-analyze-image)
- ANTS. (http://stnava.github.io/ANTs/)
- export_fig. (https://github.com/altmany/export_fig)
### Step 2: Organize your data into the following structure.
- `ratxxx` is the label for the subject. You can use other names here.
- Folder `rfmri_unprocessed` under `ratxxx` contains raw EPI scans for the subject.
```bash
├── rat001
│   ├── rfmri_unprocessed
│   │   ├── 01.nii
│   │   ├── 02.nii
│   │   ├── 03.nii
│   │   └── 04.nii
├── rat002
...
```
### Step 3: Despiking (motion scrubbing)
This step discards frames with excessive motion.
- Change `data_dir` parameter in `despiking.m` to the path to your data.
- Run `despiking.m`.
- The script will create a folder `rfmri_intermediate` in each subject folder `ratxxx` and generate the following files:
    - `xx_despiked.json`: despiking information for `xx.nii` in `rfmri_unprocessed`, containing framewise displacements, scrubbing criterion, and scrubbed frames.
    - `xx_despiked.nii.gz`: despiked fMRI scan.

- *Notes on JSON: "JSON (JavaScript Object Notation) is text, written with JavaScript object notation. It is easy for humans to read and write. It is easy for machines to parse and generate." **The `.json` files generated by this toolbox track the preprocessing steps through which the corresponding scans went.***

### Step 4: Rigid-body registration
- Use `alignment_checking_tool.m` (a graphical user interface) to manually register a rsfMRI scan to a built-in anatomical template.
- `alignment_checking_tool.m` generates the following files under `rfmri_intermediate` in each subject folder `ratxxx`:
    - `xx_tform.mat`: rigid-body registration matrix.
    - `xx_registered.json`: Add rigid-body registration matrix.
    - `xx_registered.nii.gz`: registered fMRI scan.

### Step 5: Motion correction
This step corrects motion (by registering every frame to the first one) in `xx_registered.nii.gz` using SPM built-in functions.
- Change `data_dir` parameter in `motion_correction.m` to the path to your data.
- Run `motion_correction.m`.
- `motion_correction.m` generates the following files under `rfmri_intermediate` in each subject folder `ratxxx`:
    - `xx_motioncorrected.json`: Add SPM settings for motion correction.
    - `xx_motion.json` and `xx_motion.txt`: motion parameters in .json format and text format respectively. 
    - `xx_motioncorrected.nii.gz`: motion-corrected fMRI scan.
    - `xx_motioncorrected_frame1.nii`: the 1st frame of `xx_motioncorrected.nii.gz` for spatial normalization.

### Step 6: Spatial normalization (warping)
This step corrects distortions in fMRI images by nonlinearly (deformable) registering them to a structural template.
- Change `path` parameter in `fmri_warp.sh` to the path to your data.
- Change `prefix_len` parameter in `fmri_warp.sh` to the length of your initial scan name. For example, for scan `../rfmri_unprocessed/01.nii`, `prefix_len`=2; for scan `../rfmri_unprocessed/scan1.nii`, `prefix_len`=5.
- Run `fmri_warp.sh` in terminal.
- `fmri_warp.sh` generates the following files:
    - `xx_affine.txt`: affine transformation matrix.
    - `xx_warp_field.nii.gz`: field to warp fMRI.
    - `xx_warped.nii.gz`: warped fMRI from `xx_motioncorrected.nii.gz`.

### Step 7: ICA cleaning
In this step, we will run ICA (IC=50) on individual scans (`xx_motioncorrected.nii.gz`), manually label bad components, and regress out time courses of these components.
#### Run ICA
- Change `data_dir` parameter in `ica_cleaning.m` to the path to your data.
- Adjust `FWHM_ica` parameter in `ica_cleaning.m` based on your need. It controls the strengh of spatial smoothing prior to ICA, via adjusting full-width-at-half-maximum of Gaussian kernel for spatial smoothing (default = 0.7 mm). Smoothing helps to make spatial IC maps look cleaner and easier to label. fMRI scans with high signal/contrast-to-noise ratio don't need smoothing (set `FWHM_ica` to 0). 
- `ica_cleaning.m`generates a folder `xx.gift_ica` under `rfmri_intermediate` in each subject folder `ratxxx`, which contains ICA outputs by SPM: 
    - `ica__ica.mat`: time courses and spatial maps of indepedent components.
#### Organize results of ICA
- This step copies necessary ICA results into one folder and generates figures of motion parameters, spatial maps, time courses, and frequency spectrums of ICs.
- Change `data_dir` parameter in `copy_ica_results_4labeling.m` to the path to your data.
- Change `ica_dir` parameter in `copy_ica_results_4labeling.m` to the path to which you plan to store organized ICA results.
- Run `copy_ica_results_4labeling.m`.
#### ICA labeling
- Use `ica_cleaning_view.m` (a graphical user interface) to label bad indepedent components. Please check labeling criterions here https://www.sciencedirect.com/science/article/pii/S1053811920305802#tbl1.  
- `ica_cleaning_view.m` stores labels under `[ica_dir]/labels`.
#### Copy ICA labels back to `data_dir`
- Change `data_dir` parameter in `copy_labels.m` to the path to your data.
- Change `ica_dir` parameter in `copy_labels.m` to the path where organized ICA results and labels are stored.
- Run `copy_labels.m`.
- `copy_labels.m` copies the labels to `ratxxx/rfmri_intermediate/xx.gift_ica/labels.csv`.

### Step 8: Soft regression + spatial/temporal filtering
This step 'soft' regresses bad IC components and 'hard' regresses motion parameters and white matter (WM) and cerebral spinal fluid(CSF) signals. After regression, spatial smoothing and temporal filtering will be done.
- Change `data_dir` parameter in `last_steps.m` to the path to your data.
- Change `regression_option` parameter in `last_steps.m` based on your need. 
    - 1: hard-regress average WM/CSF signal and motion parameters
    - 2: hard-regress principal components of WM/CSF signals and motion parameters
- Run `last_steps.m`.
- `last_steps.m` generates the following files under `rfmri_intermediate`:
    - `xx_WMCSF_timeseries.txt` and `xx_WMCSF_timeseries.json`: WM/CSF signals and format description respectively.
    - `xx_motion.txt`: motion parameters.
    - `xx_cleaned.nii.gz` and `xx_cleaned.json`: cleaned fMRI and its preprocessing info.
and the following files under `rfmri_processed`:
    - `xx.nii` and `xx.json`: preprocessed fMRI and its preprocessing info.

## Example folder structure after preprocessing
```bash
./
├── rat001
│   ├── rat001_info.json  [Sequence names, acquisition dates, number of frames, and corresponding names inside folders]
│   ├── raw  [Nifti files converted from raw Bruker data using Bru2Nii (https://github.com/neurolabusc/Bru2Nii)]
│   │   ├── X2P1.nii
│   │   ├── X4P1.nii
│   │   ├── X7P1.nii
│   │   ├── X8P1.nii
│   │   └── ...
│   ├── rfmri_unprocessed  [rsfMRI scans from the folder 'raw']
│   │   ├── 01.nii
│   │   ├── 02.nii
│   │   ├── 03.nii
│   │   └── 04.nii
│   ├── rfmri_intermediate  [Intermediate files generated from preprocessing] 
│   │   ├── 01_despiked.json	[Contains framewise displacements, scrubbing criterion, and scrubbed frames]
│   │   ├── 01_despiked.nii.gz  [Not further processed since more than 10% of the frames were motion-scrubbed]
│   │   ├── 02_despiked.json
│   │   ├── 02_despiked.nii.gz  [Despiked image] 
│   │   ├── 02_registered.json	[Contains rigid-body registration matrix]
│   │   ├── 02_registered.nii.gz  [Manually coregistered image] 
│   │   ├── 02_motioncorrected.json
│   │   ├── 02_motioncorrected.nii.gz  [Motion corrected image]
│   │   ├── 02_warped.json
│   │   ├── 02_warped.nii.gz  [Warped image]
│   │   ├── 02_warp_field.nii.gz  [Deformation field]
│   │   ├── 02_warp_affine.txt  [Affine transformation applied with the deformation field]
│   │   ├── 02_motion.json
│   │   ├── 02_motion.txt  [Motion parameters]
│   │   ├── 02.gift_ica  [Results from single-scan ICA]
│   │   │   ├── ica__ica_br1.mat
│   │   │   ├── ica__ica_c1-1.mat
│   │   │   ├── ica__ica.mat
│   │   │   ├── ica__icasso_results.mat
│   │   │   ├── ica_Mask.hdr
│   │   │   ├── ica_Mask.img
│   │   │   ├── ica__pca_r1-1.mat
│   │   │   ├── ica__postprocess_results.mat
│   │   │   ├── ica__results.log
│   │   │   ├── ica__sub01\_component\_ica\_s1_.mat
│   │   │   ├── ica__sub01\_component\_ica\_s1_.nii
│   │   │   ├── ica__sub01\_timecourses\_ica\_s1_.nii
│   │   │   ├── ica_Subject.mat
│   │   │   └── labels.csv  [IC labels: only the ones labeled with 'noise' were soft-regressed] 
│   │   ├── 02\_WMCSF_timeseries.json
│   │   ├── 02\_WMCSF_timeseries.txt  [Averaged signal and PCs from white matter and ventricle voxels]
│   │   ├── ...
│   └── rfmri_processed  [Preprocessed images]
│       ├── 02.json
│       ├── 02.nii
│       ├── 04.json
│       └── 04.nii
├── rat002
│   ├── ...
```
