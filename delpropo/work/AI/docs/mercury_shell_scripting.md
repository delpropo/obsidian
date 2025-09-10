Hello - made some changes - commit 2

 ## Housekeeping information
#### All DTI data are located in the following location in Mercury Server.

	nfs/corenfs/psych-mercury-data/Data/DTI/SubjectID

#### SubjectID provides information about the cohort and visit.

	NF102_2

| Notation     | meaning        |
| ------------ | -------------- |
| N            | Non-stuttering |
| F            | Female         |
| 10X          | Cohort 1       |
| SubjectID    | 102            |
| Visit Number | 2              |

#### Each participant data folder contains the following files.

| **Description**                | **Files**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| anatomical images              | adc.nii, ad.nii, anat.nii, anat_orig.nii.gz                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Output folders from Freesurfer | report, misc, surf, mri                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Secondary outputs              | fa.nii, rd.nii                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Normalized secondary outputs   | wadc.nii, wad.nii, wanat.nii, wb0.nii, wfa.nii, wFA_skeleton.nii<br>,rd.nii                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Other                          | b0_brain_mask.nii.gz, b0_brain.nii.gz, b0.nii, data.nii                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| DTI data                       | dti.nii                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Eddy correction outputs        | tmp.eddy.nii.eddy_command_txt, tmp.eddy.nii.eddy_movement_rms, tmp.eddy.nii.eddy_outlier_map, tmp.eddy.nii.eddy_outlier_n_sqr_stdev_map, tmp.eddy.nii.eddy_outlier_n_stdev_map, tmp.eddy.nii.eddy_outlier_report, tmp.eddy.nii.eddy_parameters, tmp.eddy.nii.eddy_post_eddy_shell_alignment_parameters, tmp.eddy.nii.eddy_post_eddy_shell_PE_translation_parameters, tmp.eddy.nii.eddy_restricted_movement_rms, tmp.eddy.nii.eddy_rotated_bvecs, tmp.eddy.nii.eddy_values_of_all_input_parameters, tmp.eddy.nii.gz |

## DTI CSD workflow

#### Load all required modules

	module load mrtrix
	module load fsl
	module load ANTs
	module load freesurfer
### Preprocessing Steps
#### Step 1 : File type conversion

*Note: there is a common `bval` and `bvec` file set for each cohort. Copied that from subject folder level to each subject folder. 

	DTISourceFolder="/nfs/corenfs/psych-mercury-data/Data/DTI"

	cp $DTISourceFolder/bvals Cohort1SubjectID/CohortSubjectID.bvals
	cp $DTISourceFolder/bvecs Cohort1SubjectID/CohortSubjectID.bvec

Combine raw diffusion data with `bval` and `bvec` files to use in future preprocessing steps.

	mrconvert dti.nii SubID_dwi.mif -fslgrad SubID.bvecs SubID.bvals

Checking generated MRtrix Image file (`mif`)  --> ==Quality Check 1== 

	mrinfo SubID_dwi.mif

*The output contains several pieces of information, such as the dimensions of the dataset and the voxel size, along with the commands that were used to generate the current file. Note that, since this is a 4-dimensional dataset, the last dimension is **time**; in other words, this file contains 26 volumes, each one with dimensions of 128x128x48 voxels. The last dimension of the `Voxel size` field - which in this case has a value of 13.7 - indicates the time it took to acquire each volume. This time is also called the repetition time, or TR. 

![[Pasted image 20240419105219.png]]


	mrinfo -size SubID_dwi.mif | awk '{print $4}'

*Note: The number in the 4th field of the dimensions header that corresponds to the number of time-points, or volumes, in the dataset. We then compare this with the number of bvals and bvecs by using awk to count the number of columns in each text file:

	awk '{print NF; exit}' SubID.bvecs
	awk '{print NF; exit}' SubID.bvals
	cat SubID.bvals
	cat SubID.bvecs
#### Step 2: Initial preprocessing (noise removal)

	dwidenoise SubID_dwi.mif SubID_dwi_den.mif -noise SubID_dwi_noise.mif 
	mrcalc SubID_dwi.mif SubID_dwi_den.mif -subtract SubID_dwi_residual.mif

Checking generated MRtrix Image file (`mif`)  --> ==Quality Check 2== 

*One quality check is to see whether the residuals load onto any part of the anatomy. If they do, that may indicate that the brain region is disproportionately affected by some kind of artifact or distortion. 

	mrview subID_dwi_residual.mif

	
![[Pasted image 20240418173006.png]]
*It is common to see a grey outline of the brain, as in the figure above. However, everything within the grey matter and white matter should be relatively uniform and blurry; if you see any clear anatomical landmarks, such as individual gyri or sulci, that may indicate that those parts of the brain have been corrupted by noise. If that happens, you can increase the extent of the denoising filter from the default of 5 to a larger number, such as 7; e.g.,

	dwidenoise your_data.mif your_data_denoised_7extent.mif -extent 7 -noise noise.mif

* NOTE: We are not doing Gibbs ringing artifact removal (It was not a general issue for this dataset) OR Extracting the Reverse Phase-Encoded Images (because we do not have two encoding directions for this dataset)
#### Step 3: Putting it all together --> preprocessing with dwipreproc

FSL commands are run so we will need to load module FSL

	module load fsl
	dwifslpreproc subID_dwi_den.mif subID_dwi_den_preproc.mif -nocleanup -pe_dir PA -rpe_none -eddy_options " --slm=linear"

The first arguments are the input and output; the second option, `-nocleanup`, will keep the temporary processing folder which contains a few files we will examine later. `-pe_dir AP` signalizes that the primary phase-encoding direction is anterior-to-posterior, and `-rpe_pair` combined with the `-se_epi` options indicates that the following input file (i.e., “b0_pair.mif”) is a pair of spin-echo images that were acquired with reverse phase-encoding directions. Lastly, `-eddy_options` specifies options that are specific to the FSL command `eddy`. You can visit the [eddy user guide](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/eddy/UsersGuide) for more options and details about what they do. For now, we will only use the options `--slm=linear` (which can be useful for data that was acquired with less than 60 directions) and `--data_is_shelled` (which indicates that the diffusion data was acquired with multiple b-values).

This command can takes roughly 2 hours. When it has finished, examine the output to see how eddy current correction and unwarping have changed the data; ideally, you should see more signal restored in regions such as the orbitofrontal cortex, which is particularly susceptible to signal dropout:

	mrview subID_dwi_den_preproc.mif -overlay.load SubID_dwi.mif

This command will display the newly preprocessed data, with the original diffusion data overlaid on top of it and colored in red. To see how the eddy currents were unwarped, open the Overlays tab and click on the box next to the image `subID_dwi.mif`. You should see a noticeable difference between the two images, especially in the frontal lobes of the brain near the eyes, which are most susceptible to eddy currents.

#### Step 4: Checking for corrupt slices  --> ==Quality Check 3==

One of the options in the `dwifslpreproc` command, “-nocleanup”, retained a directory with the string “tmp” in its title. Within this folder is a file called `dwi_post_eddy.eddy_outlier_map`, which contains strings of 0’s and 1’s. Each 1 represents a slice that is an outlier, either because of too much motion, eddy currents, or something else.

The following code, run from the `dwi` directory, will navigate into the “tmp” folder and calculate the percentage of outlier slices:

	cd dwifslpreproc-tmp-*
	totalSlices=`mrinfo dwi.mif | grep Dimensions | awk '{print $6 * $8}'`
	totalOutliers=`awk '{ for(i=1;i<=NF;i++)sum+=$i } END { print sum }' dwi_post_eddy.eddy_outlier_map`
	echo "If the following number is greater than 10, you may have to discard this subject because of too much motion or corrupted slices"
	echo "scale=5; ($totalOutliers / $totalSlices * 100)/1" | bc | tee percentageOutliers.txt
	cd ..

#### Step 5: Generating a Mask

Create a mask to restrict your analysis only to brain voxels; this will speed up the rest of your analyses.

To do that, it can be useful to run a command beforehand called `dwibiascorrect`. This can remove inhomogeneities detected in the data that can lead to a better mask estimation. However, it can in some cases lead to a worse estimation; as with all of the preprocessing steps, you should check it before and after each step:

	module load ANTs
	dwibiascorrect ants subID_dwi_den_preproc.mif subID_den_preproc_unbiased.mif -bias bias.mif


You are now ready to create the mask with `dwi2mask`, which will restrict your analysis to voxels that are located within the brain:

	dwi2mask subID_den_preproc_unbiased.mif mask.mif
	
OR force overwrite mask.mif that already exist

	dwi2mask -force subID_den_preproc_unbiased.mif mask.mif

Check the output of this command by typing:

	mrview mask.mif

Make sure the mask does not have any holes! Fix the holes with following commands, if there are any.

	mrconvert subID_den_preproc_unbiased.mif subID_unbiased.nii
	bet2 subID_unbiased.nii subID_masked -m -f 0.2
	mrconvert subID_masked_mask.nii.gz mask_new.mif


### Constrained Spherical Deconvolution Analysis

#### Step 1 : Fiber Orientation Distribution
##### Response function estimation : Estimate different response functions for different tissue types (WM, GM, CSF)

In order to generate streamlines, first we need to estimate the orientation of fiber(s) in each voxel. We do this using Constrained Spherical Deconvolution (CSD; Tournier et al., 2004, 2007), instead with the tensor model, which was shown to outperform the performance of DTI in regions of crossing/kissing fibers (Farquharson et al., 2013). To perform CSD< response function (RF) is necessary, which is used as a kernal for deconvolution. 

As many voxels contain both white and grey matter, or white matter and CSF (partial volumes), CSD is flawed in such voxels. We can improve our results by estimating different RFs for different tissue types. (i.e., The RF in white matter models the signal which is expected if there was only a fiber bundle with one coherent orientation present in a voxel). 

The RF generation is best done with DW data with different b-values as different b values are sensitive to different tissue types. This idea is at the core of multi-shell multi-tissue CSD (MSMT; Jeurissen et al., 2014). Our dataset has two b values (0 and 1000). Thus we can attempt this. However, as there are only two b values, we are only going to get RF for two tissue types (WM, and GM). This is a limitation as we do not model CSF well.

However, the code requires that a csf output is provided --> thus we left  the csf.txt output in the command. 

	dwi2response dhollander subID_dwi_den_preproc.mif -voxels voxels.mif wm.txt gm.txt csf.txt

This dataset can be viewed by typing the following:

	mrview subID_dwi_den_preproc_unbiased.mif -overlay.load voxels.mif

The output from the `dwi2response` command, showing which voxels were used to construct a basis function for each tissue type. Red: CSF voxels; Green: Grey Matter voxels; Blue: White Matter voxels. Make sure that these colors are located where they should be; for example, the red voxels should be within the ventricles.

You can then check the response function for each tissue type by typing:

	shview wm.txt
	shview gm.txt
	shview csf.txt

Now we are ready do run the FOD analysis

	dwi2fod msmt_csd subID_den_preproc.mif -mask mask.mif wm.txt wmfod.mif gm.txt gmfod.mif csf.txt csffod.mif
	mrconvert -coord 3 0 wmfod.mif - | mrcat csffod.mif gmfod.mif - vf.mif
	mrview vf.mif -odf.load_sh wmfod.mif


Also trying single shell approach --> This commands takes a while at the 3 iterations. SO be patient.

	module load mrtrix/tissue

	ss3t_csd_beta1 sub-01_den_preproc_unbiased.mif wm.txt wmfod.mif gm.txt gmfod.mif csf.txt csffod.mif -mask mask.mif

Example: 

	ss3t_csd_beta1 NF130_1_den_preproc_unbiased.mif wm.txt wmfod_ss3t.mif gm.txt gmfod_ss3t.mif csf.txt csffod_ss3t.mif -mask mask.mif

	mrconvert -coord 3 0 wmfod_ss3t.mif - | mrcat csffod_ss3t.mif gmfod_ss3t.mif - vf_ss3t.mif


	mrview vf_ss3t.mif -odf.load_sh wmfod_ss3t.mif


SS3T version is only slightly different from MSMT approach (but you can see more green - csf areas)--> MSMT approach with CSF tissue ditched is very sparse and possibly leads to reduced number of fiber tracks at streamline step we noted earlier. Here is a comparison. SS3T vs MSMT (without CSF) vs MSMT (with CSF)

![[Pasted image 20240419152546.png]]

#### Normalization

Later on, for group-level analysis with the data that has been generated for each subject., we will need to **normalize** the FODs. This ensures that any differences we see are not due to intensity differences in the image, similar to how we correct for the size of the brain when comparing volumetric differences across subjects.

To normalize the data, we will use the `mtnormalise` command. This requires an input and output for each tissue type, as well as a mask to restrict the analysis to brain voxels:

	mtnormalise wmfod.mif wmfod_norm.mif gmfod.mif gmfod_norm.mif csffod.mif csffod_norm.mif -mask mask.mif

Noted that we get an error when trying with MSMT approach with csf.txt

![[Pasted image 20240419152000.png]]
Thus, switching to SS3T method output FODs:
	
	mtnormalise wmfod_ss3t.mif wmfod__ss3t_norm.mif gmfod_ss3t.mif gmfod_ss3t_norm.mif csffod_ss3t.mif csffod_ss3t_norm.mif -mask mask.mif


### Generating Tissue boundaries

For streamline analysis, **seeds** will be placed at random locations along the boundary between the grey matter and the white matter. A streamline will grow from each seed and trace a path from that seed region until it terminates in another region. Some of the streamlines will terminate in places that don’t make sense - for example, a streamline may terminate at the border of the ventricles. We will cull these “error” streamlines, and be left with a majority of streamlines that appear to connect distant grey matter regions.

To do this, we will first need to create a **boundary** between the grey matter and the white matter. The MRtrix command `5ttgen` will use FSL’s FAST, along with other commands, to segment the anatomical image into five tissue types:

	1. Grey Matter;
	2. Subcortical Grey Matter (such as the amygdala and basal ganglia);
	3. White Matter;
	4. Cerebrospinal Fluid; and
	5. Pathological Tissue. 

Once we have segmented the brain into those tissue classes, we can then use the boundary as a mask to restrict where we will place our seeds.

#### Step 1 : Convert anatomical image to MRtrix format

the anatomical image is present in each subject folder as `anat_orig.nii.gz` --> we will make a copy as `T1.ni.gz` --> unzip it (`TI.nii`) and convert it to `T1.mif` 

	cp anat_orig.nii.gz T1.nii.gz
	gzip -d T1.nii.gz
	mrconvert T1.nii T1.mif

#### Step 2: Segmenting to tissue types

We will now use the command `5ttgen` to segment the anatomical image into the tissue types listed above: (~15 mins)

	5ttgen fsl T1.mif 5tt_nocoreg.mif
	mrview 5tt_nocoreg.mif

If the segmentation has finished successfully, you should see the following images when you type `mrview 5tt_nocoreg.mif` (pressing the left and right arrow keys scrolls through the different tissue types): We only see four types - not five. The final volume will be blank as we do not have pathological Tissue.

![[Pasted image 20240419161022.png]]
### Co-registering the Diffusion and Anatomical Images

Next step is to co-register the anatomical and diffusion-weighted images. This ensures that the boundaries of the tissue types are aligned with the boundaries of the diffusion-weighted images; even small differences in the location of the two scans can throw off the tractography results.

We will first use the commands `dwiextract` and `mrmath` to average together the B0 images from the diffusion data. These are the images that look most like T2-weighted functional scans, since a diffusion gradient wasn’t applied during their acquisition - in other words, they were acquired with a b-value of zero. 

	dwiextract subID_den_preproc_unbiased.mif - -bzero | mrmath - mean mean_b0.mif -axis 3

There are two parts to this command, separated by a pipe (”`|`”). The left half of the command, `dwiextract`, takes the preprocessed diffusion-weighted image as an input, and the `-bzero` option extracts the B0 images; the solitary `-` argument indicates that the output should be used as input for the second part of the command, to the right of the pipe. `mrmath` then takes these output B0 images and computes the mean along the 3rd axis, or the time dimension. In other words, if we start with an index of 0, then the number 3 indicates the 4th dimension, which simply means to average over all of the volumes.

In order to carry out the coregistration between the diffusion and anatomical images, we will need to take a brief detour outside of MRtrix. The software package doesn’t have a coregistration command in its library, so we will need to use another software package’s commands instead. Although you can choose any one you want, we will focus here on FSL’s `flirt` command.

The first step is to convert both the segmented anatomical image and the B0 images we just extracted:

	mrconvert mean_b0.mif mean_b0.nii.gz
	mrconvert 5tt_nocoreg.mif 5tt_nocoreg.nii.gz

Since `flirt` can only work with a single 3D image (not 4D datasets), we will use `fslroi` to extract the first volume of the segmented dataset, which corresponds to the Grey Matter segmentation:

	fslroi 5tt_nocoreg.nii.gz 5tt_vol0.nii.gz 0 1

We then use the `flirt` command to coregister the two datasets:

	flirt -in mean_b0.nii.gz -ref 5tt_vol0.nii.gz -interp nearestneighbour -dof 6 -omat diff2struct_fsl.mat

This command uses the grey matter segmentation (i.e., “5tt_vol0.nii.gz”) as the reference image, meaning that it stays stationary. The averaged B0 images are then moved to find the best fit with the grey matter segmentation. The output of this command, “diff2struct_fsl.mat”, contains the **transformation matrix** that was used to overlay the diffusion image on top of the grey matter segmentation.

Now that we have generated our transformation matrix, we will need to convert it into a format that can be read by MRtrix. That is, we are now ready to travel back into MRtrix after briefly stepping outside of it. The command `transformconvert` does this:

	transformconvert diff2struct_fsl.mat mean_b0.nii.gz 5tt_nocoreg.nii.gz flirt_import diff2struct_mrtrix.txt

Note that the above steps used the anatomical segmentation as the reference image. We did this because usually the coregistration is more accurate if the reference image has higher spatial resolution and sharper distinction between the tissue types. However, we also want to introduce as few edits and interpolations to the functional data as possible during preprocessing. Therefore, since we already have the steps to transform the diffusion image to the anatomical image, we can take the inverse of the transformation matrix to do the opposite - i.e., coregister the anatomical image to the diffusion image:

	mrtransform 5tt_nocoreg.mif -linear diff2struct_mrtrix.txt -inverse 5tt_coreg.mif

The resulting file, “5tt_coreg.mif”, can be loaded into `mrview` in order to examine the quality of the coregistration:

	mrview subID_den_preproc_unbiased.mif -overlay.load 5tt_nocoreg.mif -overlay.colourmap 2 -overlay.load 5tt_coreg.mif -overlay.colourmap 1

The “overlay.colourmap” options specify different color codes for each image that is loaded. In this case, the boundaries before coregistration will be depicted in blue, and the boundaries after coregistration will be shown in red. The change might be slight but makes a big differences for later analysis steps.

### Create seed boundaries

The last step to create the “seed” boundary - the boundary separating the grey from the white matter, which we will use to create the seeds for our streamlines - is created with the command `5tt2gmwmi` (which stands for “5 Tissue Type (segmentation) to Grey Matter / White Matter Interface)

	5tt2gmwmi 5tt_coreg.mif gmwmSeed_coreg.mif

Again, we will check the result with `mrview` to make sure the interface is where we think it should be:

	mrview subID_den_preproc_unbiased.mif -overlay.load gmwmSeed_coreg.mif


## Streamlines

### Anatomically Constrained Tractography

One of MRtrix’s features is **Anatomically Constrained Tractography**, or ACT. This method will only determine that a streamline is valid if it is biologically plausible. For example, a streamline that terminates in the cerebrospinal fluid will be discarded, since white matter tracts tend to both originate and terminate in grey matter. In other words, the streamlines will be constrained to the white matter. Anatomically constrained tractography isn’t a separate preprocessing step, but rather an option that can be included with the command `tckgen`, which generates the actual streamlines.

#### Generating Streamlines with tckgen

MRtrix is able to do both **deterministic** and **probabilistic** tractography. In deterministic tractography, the direction of the streamline at each voxel is determined based on the predominant fiber orientation; in other words, the streamline is determined by a single parameter. MRtrix includes multiple options to do this type of deterministic tractography, such as `FACT` or `tensor_det`.

The other method, probabilistic tractography, is the default in MRtrix. In this approach, multiple streamlines are generated from seed regions all along the boundary between the grey matter and white matter. The direction of the streamline will most likely follow the predominant fiber orientation density, but not always; due to a large number of samples, some streamlines will follow other directions. This becomes less likely if the FOD is extremely strong in one direction - for example, the FODs within a structure such as the corpus callosum will tend to all be aligned left-to-right - but the sampling becomes more diverse in regions that do not have a predominant fiber orientation.

The default method is to use an algorithm known as iFOD2, which will use a probabilistic streamline approach. Other algorithms can be found at [this site](https://mrtrix.readthedocs.io/en/latest/reference/commands/tckgen.html). We will use the default of iFOD2.

#### How Many Streamlines?

There is a trade-off between the number of generated streamlines and the amount of time that it takes. More streamlines result in a more accurate reconstruction of the underlying white-matter tracts, but estimating a large number of them can take a prohibitively long time.

The “correct” number of streamlines to use is still being debated, but at least ==10 million== or so should be a good starting place:

	tckgen -act 5tt_coreg.mif -backtrack -seed_gmwmi gmwmSeed_coreg.mif -nthreads 8 -maxlength 250 -cutoff 0.06 -select 10000000 wmfod_ss3t_norm.mif tracks_10M.tck

In this command, 

| -act               | specifies that we will use the anatomically-segmented image to constrain our analysis to the white matter.                                                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -backtrack         | indicates for the current streamline to go back and run the same streamline again if it terminates in a strange place (e.g., the cerebrospinal fluid).                                                                                                  |
| -maxlength         | sets the maximum tract length, in voxels, that will be permitted; <br>and “-cutoff” specifies the FOD amplitude for terminating a tract (for example, a value of 0.06 would not permit a streamline to go along an FOD that is lower than that number). |
| -seed_gmwmi        | takes as an input the grey-matter / white-matter boundary that was generated using the `5tt2gmwmi` command.                                                                                                                                             |
| -nthreads          | specifies the number of processing cores you wish to use, in order to speed up the analysis.                                                                                                                                                            |
| -select            | indicates how many total streamlines to generate. Note that a shorthand can be used if you like; instead of, say, 10000000, you can rewrite it as 10000k (meaning “ten thousand thousands”, which equals “ten million”).                                |
| last two arguments | The last two arguments specify both the input (`wmfod_norm.mif`) and a label for the output (`tracks_10M.tck`).                                                                                                                                         |

If you want to visualize the output, extract a subset of the output by using `tckedit`:

	tckedit tracks_10M.tck -number 200k smallerTracks_200k.tck

This can then be loaded into `mrview` by using the “-tractography.load” option, which will automatically overlay the smallerTracks_200k.tck file onto the preprocessed diffusion-weighted image:

	mrview subID_den_preproc_unbiased.mif -tractography.load smallerTracks_200k.tck

The streamlines should be constrained to the white matter, and they should be color-coded appropriately. For example, the corpus callosum should be mostly red, and the corona radiata should be mostly blue.

#### Refining the streamlines

Although we have created a diffusion image with reasonable streamlines, also known as a **tractogram**, we still have a problem with some of the white matter tracts being over-fitted, and others being under-fitted. This can be addressed with the `tcksift2` command.

The reason is that some tracts will be threaded with more streamlines than others, because the fiber orientation densities are much clearer and more attractive candidates for the probabilistic sampling algorithm that was discussed above. In other words, certain tracts can be over-represented by the amount of streamlines that pass through them not necessarily because they contain more fibers, but because the fibers tend to all be orientated in the same direction.

To counter-balance this overfitting, the command `tcksift2` will create a text file containing weights for each voxel in the brain:

	tcksift2 -act 5tt_coreg.mif -out_mu sift_mu.txt -out_coeffs sift_coeffs.txt -nthreads 8 tracks_10M.tck wmfod_ss3t_norm.mif sift_1M.txt

The output from the command, “sift_1M.txt”, can be used with the command `tck2connectome` to create a matrix of how much each ROI is connected with every other ROI in the brain - a figure known as a **connectome** - which will weight each ROI. 

## Creating and Viewing the Connectome

We can create a **connectome** that represents the number of streamlines connecting different parts of the brain. To do that, we have to first parcellate the brain into different regions, or nodes. One way to do this is by using an **atlas**, which assigns each voxel in the brain to a specific ROI.

We will be using the atlases that come with [FreeSurfer](https://andysbrainbook.readthedocs.io/en/latest/FreeSurfer/FS_ShortCourse/FS_11_ROIAnalysis.html#fs-11-roianalysis). Accordingly, our first step will be to run the subject’s anatomical image through recon-all, which you can read more about [here](https://andysbrainbook.readthedocs.io/en/latest/FreeSurfer/FS_ShortCourse/FS_03_ReconAll.html#fs-03-reconall):

	module load freesurfer
	recon-all -i anat_orig.nii.gz -s subID_recon -all
	
We can also use the T1.nii we generated earlier through anat-Orign.nii.gz for this step instead.

	recon-all -i T1.nii -s subID_recon -all


This will take a few hours (4.425 hours!), depending on the speed of your computer. When it has finished, make sure to check the output by using the QA procedures described in [this chapter](https://andysbrainbook.readthedocs.io/en/latest/FreeSurfer/FS_ShortCourse/FS_12_FailureModes.html#fs-12-failuremodes).
### Creating the Connectome

When recon-all has finished, we will need to convert the labels of the FreeSurfer parcellation to a format that MRtrix understands. The command `labelconvert` will use the parcellation and segmentation output of FreeSurfer to create a new parcellated file in .mif format:

Define freesurfer and mrtrix_home path to make the code generic.

$FREESURFER_HOME='/app/apps/rhel8/freesurfer/7.1.1'  
$MRTRIX_HOME='/app/apps/rhel8/mrtrix/3.0.4'

	labelconvert ../subjects/subID_recon_all/mri/aparc+aseg.mgz $FREESURFER_HOME/FreeSurferColorLUT.txt $MRTRIX_HOME/share/mrtrix3/labelconvert/fs_default.txt subID_parcels.mif
	
We then need to create a whole-brain connectome, representing the streamlines between each parcellation pair in the atlas (in this case, 84x84). 

| symmetric        | option will make the lower diagonal the same as the upper diagonal       |
| ---------------- | ------------------------------------------------------------------------ |
| scale_invnodevol | option will scale the connectome by the inverse of the size of the node. |
|                  |                                                                          |

	tck2connectome -symmetric -zero_diagonal -scale_invnodevol -tck_weights_in sift_1M.txt tracks_1M.tck subID_parcels.mif subID_parcels.csv -out_assignment assignments_subID_parcels.csv

Used tracks_10M.tck ?? is that wrong ?? no tracks_1M.tck file available
### Viewing the Connectome

Once you have created the `parcels.csv` file, you can view it as a matrix in `Matlab`. First, you will need to import it:

	module load MATLAB/R2021a
	
	connectome = importdata('NF130_1_parcels.csv');

And then you will need to view it as a scaled image, so that higher structural connectivity pairs are brighter:

	imagesc(connectome);

![[Pasted image 20240420093515.png]]

The most noticeable feature is a division of the figure into two distinct “boxes”, representing increased structural connectivity within each hemisphere. You will also observe a relatively brighter line traced along the diagonal, representing higher structural connectivity between nearby nodes. Brighter boxes in the opposing bottom-left and upper-right corners represent increased structural connectivity between homologous regions.

To make these associations more obvious, you can change the scaling of the color map:

	imagesc(connectome, [0 1]);

![[Pasted image 20240420093550.png]]

### Generating ROI based tractography 

To figure out the ROI look at fs_default.txt in MTRIX folder that was used in the label-convert step:

`cat $MRTRIX_HOME/share/mrtrix3/labelconvert/fs_default.txt`

For IFG:

	connectome2tck -nodes 17 tracks_10M.tck assignments_NF130_1_parcels.csv -files per_node LeftParsopecularis
	mrview 5tt_coreg.mif -tractography.load LeftParsopecularis17.tck  


Initially, using MSMT --> here were reduced tract density and many tracts were missing

![[Pasted image 20240421213931.png]]![[Pasted image 20240421214004.png]]
![[Pasted image 20240421214013.png]]
Using SH3T --> the results were much better (more fibers reaching the posterior auditory/parietal regions)

1Mtracks

![[Pasted image 20240422173655.png]]
![[Pasted image 20240422173637.png]]

![[Pasted image 20240422173612.png]]


10Mtracks
![[Pasted image 20240422173937.png]]
![[Pasted image 20240422173905.png]]

![[Pasted image 20240422173845.png]]

Go back and generate the 1M track version all the way from `tckgen` step if you want to use a different number of tracks for the process. For viewing, you can always reduce to 200k tracks using `tckedit`. 

	tckgen -act 5tt_coreg.mif -backtrack -seed_gmwmi gmwmSeed_coreg.mif -nthreads 8 -maxlength 250 -cutoff 0.06 -select 1000000 wmfod_ss3t.mif tracks_1M.tck
	
	tckedit tracks_1M.tck -number 200k smallerTracks_200k.tck
	
	mrview NF130_1_dwi_den_preproc.mif -tractography.load smallerTracks_200k.tck

	tcksift2 -act 5tt_coreg.mif -out_mu sift_mu.txt -out_coeffs sift_coeffs.txt -nthreads 8 tracks_1M.tck wmfod_norm.mif sift_1M.txt

--------------------------------------------------------------------------

	tck2connectome -symmetric -zero_diagonal -scale_invnodevol -tck_weights_in sift_1M.txt tracks_1M.tck NF101_1_parcels.mif NF101_1_parcels_1Mtrack.csv -out_assignment assignments_NF101_1_parcels_1Mtrack.csv


### Comparing different track densities and single shell multi shell methods

MMMT

Initially, using MSMT --> here were reduced tract density and many tracts were missing

1M tracks
![[Pasted image 20240421213931.png]]

SS3T
1Mtracks

![[Pasted image 20240422173655.png]]


10Mtracks
![[Pasted image 20240422173937.png]]


### Subcortical segmentations

Freesurfer V3 provides more detailed subcortical segmentations (run time 11 hours)

https://surfer.nmr.mgh.harvard.edu/fswiki/FreeSurferVersion3

In automatic subcortical segmentation, each voxel in the normalized brain volume is assigned one of about 40 labels, including:

- Cerebral White Matter, Cerebral Cortex, Lateral Ventricle, Inferior Lateral Ventricle, Cerebellum White Matter, Cerebellum Cortex, Thalamus, Caudate, Putamen, Pallidum, Hippocampus, Amygdala, Lesion, Accumbens area, Vessel, Third Ventricle, Fourth Ventricle, Brain Stem, Cerebrospinal Fluid

[FreeSurfer](https://surfer.nmr.mgh.harvard.edu/fswiki/FreeSurfer) now runs automated labeling of the brain volume and this is included in the stable v3.0 release, [during the -autorecon2 stage](https://surfer.nmr.mgh.harvard.edu/fswiki/ReconAllDevTable). However, if you processed your anatomical data using previous versions and you wish to obtain the automated labels, you can just run the subcortical segmentation separately.

To obtain automatically segmented volumes for the first time (assuming the [-autorecon1 stage](https://surfer.nmr.mgh.harvard.edu/fswiki/ReconAllDevTable) has completed), run:

Make sure `$SUBJECT_DIR` is set accordingly 
`SUBJECTS_DIR='/nfs/corenfs/psych-mercury-data/Data/DTI/subjects'`
`recon-all -subcortseg -subjid <subject name>'

`recon-all -subcortseg -subjid ../NF130_1_recon`

>>>> Ventrolateral thalamus --> also check Frank's model
>>>> For basal ganglia thalamo cortical loop
>>>> 
### Generating batch scripts

For now, we will do this for a single subject. The scripts will be written in four parts:

1. The first script will perform all of the preprocessing, from denoising to `tcksift2`;
    
2. The second script will perform QA checks for each of the major preprocessing outputs;
    
3. The third script will preprocess the structural images using `recon-all`; and
    
4. The last script will create the connectome.
    
`recon-all` isn’t part of the MRtrix pipeline _per se_ - you can use any atlas you want, and you are not restricted to FreeSurfer - but we will include it as a prerequisite for creating the connectome.

[[batch script (Based on Andy's scripts) - modified]]





	#!/bin/bash
	
	mrconvert dti.nii DTI.mif -fslgrad NF101_1.bvecs NF101_1.bvals
	dwidenoise DTI.mif DTI_den.mif -noise DTI_noise.mif
	mrcalc DTI.mif DTI_den.mif -subtract DTI_residual.mif
	mrview DTI_residual.mif
	dwifslpreproc DTI_den.mif DTI_den_preproc.mif -nocleanup -pe_dir PA -rpe_none -eddy_options " --slm=linear"
	mrview DTI_den_preproc.mif
	dwi2mask DTI_den_preproc.mif DTI_mask.mif
	mrview DTI_mask.mif 
	dwi2response dhollander DTI_den_preproc.mif -voxels voxels.mif wm.txt gm.txt csf.txt
	shview wm.txt
	shview gm.txt
	shview csf.txt
	dwi2fod msmt_csd DTI_den_preproc.mif -mask DTI_mask.mif wm.txt wmfod.mif gm.txt gmfod.mif csf.txt csffod.mif
	mrconvert -coord 3 0 wmfod.mif - | mrcat csffod.mif gmfod.mif - vf.mif
	mrview vf.mif -odf.load_sh wmfod.mif

	mrconvert T1.nii T1.mif
	5ttgen fsl T1.mif 5tt_nocoreg.mif
	mrview 5tt_nocoreg.mif

	dwiextract DTI_den_preproc.mif - -bzero | mrmath - mean mean_b0.mif -axis 3
	mrconvert mean_b0.mif mean_b0.nii.gz
	mrconvert 5tt_nocoreg.mif 5tt_nocoreg.nii.gz
	fslroi 5tt_nocoreg.nii.gz 5tt_vol0.nii.gz 0 1
	flirt -in mean_b0.nii.gz -ref 5tt_vol0.nii.gz -interp nearestneighbour -dof 6 -omat diff2struct_fsl.mat
	transformconvert diff2struct_fsl.mat mean_b0.nii.gz 5tt_nocoreg.nii.gz flirt_import diff2struct_mrtrix.txt
	mrtransform 5tt_nocoreg.mif -linear diff2struct_mrtrix.txt -inverse 5tt_coreg.mif
	mrview DTI_den_preproc.mif -overlay.load 5tt_nocoreg.mif -overlay.colourmap 2 -overlay.load 5tt_coreg.mif -overlay.colourmap 1
	5tt2gmwmi 5tt_coreg.mif gmwmSeed_coreg.mif
	mrview DTI_den_preproc.mif -overlay.load gmwmSeed_coreg.mif

	tckgen -act 5tt_coreg.mif -backtrack -seed_gmwmi gmwmSeed_coreg.mif -nthreads 8 -maxlength 250 -cutoff 0.06 -select 1000000 wmfod.mif tracks_1M.tck
	tckedit tracks_1M.tck -number 200k smallerTracks_200k.tck
	mrview DTI_den_preproc.mif -tractography.load smallerTracks_200k.tck

	tcksift2 -act 5tt_coreg.mif -out_mu sift_mu.txt -out_coeffs sift_coeffs.txt -nthreads 8 tracks_1M.tck wmfod_norm.mif sift_1M.txt

	recon-all -i T1.nii -s sub-NF101_1_recon -all

	labelconvert aparc+aseg.mgz $FREESURFER_HOME/FreeSurferColorLUT.txt /app/apps/rhel8/mrtrix/3.0.4/share/mrtrix3/labelconvert/fs_default.txt NF101_1_parcels.mif

	tck2connectome -symmetric -zero_diagonal -scale_invnodevol -tck_weights_in sift_1M.txt tracks_1M.tck NF101_1_parcels.mif NF101_1_parcels.csv -out_assignment assignments_NF101_1_parcels.csv


Matlab commands
connectome = importdata('NF101_1_parcels.csv');



### Code to select ROIs and seeds of interest

mrcalc NF101_1_parcels.mif 38 -eq L_PU.mif
mrcalc NF101_1_parcels.mif 45 -eq R-PU.mif


connectome2tck -nodes 38,45 -exclusive tracks_1M.tck assignments_NF101_1_parcels.csv moto
mrview 5tt_coreg.mif -tractography.load moto38-45.tck

connectome2tck -nodes 36,38 -exclusive tracks_1M.tck assignments_NF101_1_parcels.csv moto

mrview 5tt_coreg.mif -tractography.load moto36-38.tck

connectome2tck -nodes 37,23 tracks_1M.tck assignments_NF101_1_parcels.csv -files per_node LeftCaudate
mrview 5tt_coreg.mif -tractography.load LeftCaudate23.tck    
            
LeftCaudate37.tck  

less $FREESURFER_HOME/FreeSurferColorLUT.txt

freeview -v 004/mri/orig.mgz \
004/mri/aparc+aseg.mgz:colormap=lut:opacity=0.4 \
-f 004/surf/lh.white:annot=aparc.annot

freeview -v mri/orig.mgz \
mri/aparc+aseg.mgz:colormap=lut:opacity=0.4 \
-f surf/lh.white:annot=aparc.annot

connectome2tck -nodes 17 tracks_1M.tck assignments_NF101_1_parcels.csv -files per_node LeftParsopecularis
mrview 5tt_coreg.mif -tractography.load LeftParsopecularis17.tck    



Generating stramlines that arandomly seeded from a mask ROI to the rest of the brain

**  

tckgen -seed_image mask_Left_Cluster1.mif tracks_1M.tck output_Left-Cluster1.tck

**