# LROC_NAC_stereo of Marius Hills skylight 

This is to set up a workflow for generating decent quality LROC NAC stereo DEMs
```
wget http://lroc.sese.asu.edu/data/LRO-L-LROC-2-EDR-V1.0/LROLRC_0027/DATA/ESM2/2016152/NAC/M1219322049LE.IMG
wget http://lroc.sese.asu.edu/data/LRO-L-LROC-2-EDR-V1.0/LROLRC_0027/DATA/ESM2/2016152/NAC/M1219322049RE.IMG
wget http://lroc.sese.asu.edu/data/LRO-L-LROC-2-EDR-V1.0/LROLRC_0027/DATA/ESM2/2016152/NAC/M1219329084RE.IMG
wget http://lroc.sese.asu.edu/data/LRO-L-LROC-2-EDR-V1.0/LROLRC_0027/DATA/ESM2/2016152/NAC/M1219329084LE.IMG
```

we use the ASP python wrapper to do the job of processing and mosaicking the NAC pieces

```
lronac2mosaic.py M1219322049LE.IMG M1219322049LE.IMG 
lronac2mosaic.py M1219329084RE.IMG M1219329084LE.IMG
```

Then we take the outputs (the final stereo pair) and use the second wrapper to create level2 which seems more reliable according to ASP manual

```
cam2map4stereo.py M1219322049LE.lronaccal.lronacecho.noproj.mosaic.norm.cub M1219329084LE.lronaccal.lronacecho.noproj.mosaic.norm.cub
```

We then crop them in order to save time

```
crop from=M1219322049LE.map.cub to=M1219322049LE.map.crop.cub sample=2200 nsamples=5000 line=30000 nlines=5000
crop from=M1219329084LE.map.cub to=M1219329084LE.map.crop.cub sample=2200 nsamples=5000 line=30000 nlines=5000
```

And we perform bundle adjustmend on the cropped images which is faster
```
bundle_adjust M1219322049LE.map.crop.cub M1219329084LE.map.crop.cub --min-matches 1 -o run_ba/run
```
Then we run stereo
```
stereo M1219322049LE.map.crop.cub M1219329084LE.map.crop.cub run_adjust/run --subpixel-mode 2 --bundle-adjust-prefix run_ba/run -—threads 16
```


PC_Align needs to align the first obtained point cloud to a reference DEM. We chose the ~60 m esolution LOLA+Kaguya merged DEM. For the sake of this analysis we will crop it on a broader region:

```
gdal_translate -projwin -1733458.7644640056 438880.4473252905 -1709916.8672144127 419142.4950550104 -of GTiff DEM_Kaguya.tif LOLA+Kaguya_crop.tif
```
```
pc_align --max-displacement 200 run_adjust/run-PC.tif LOLA+Kaguya_crop.tif --highest-accuracy --save-transformed-source-points -o spheroid_DEM
```

We now interpolate the DEM as a first initial guess for shape from shading
```
point2dem -r moon run_full2/run-PC.tif --t_srs "+proj=eqc +lat_0=0 +lon_0=0 +a=1737400 +b=1737400 +units=m +no_defs" --nodata -32767 --dem-hole-fill-len 100 --orthoimage-hole-fill-len 100 --remove-outliers-params 75.0 3.0 --orthoimage run_full2/run-L.tif -o run_full2/ortho-L
```

**Now the funny part begins**

Shape from shading is very computationally heavy procedure and we need to set it up properly.
Parallelized sfs is the fastest way. Before we need to install gnu parallel

On linux machines: 

```
sudo apt-get install parallel
```

On Mac OS Homebrew has to be installed:

```
brew install parallel
```


Before proceeding we need to ensure that if we are running python 3 on our machine we modify the parallel_sfs.py program located into the ASP installation folder at StereoPipeline-2.6.2-2019-06-17-x86_64-OSX/libexec/parallel_sfs

we need to change in line 347 from
```
argumentFile     = file(argumentFilePath, 'w')
```

into 
```
argumentFile     = open(argumentFilePath, 'w')
```
if we have only python 2 on our machine we can skip this step


**Shape from Shading**

Shape from shading relies intensively on ram but is basically run as single threaded process. With this wrapper we can split the input DEM in tiles (with a certain pixel overlap) and parallelize shape from shading for each tile. Eventually they will be composited into the final result. This saves time and resources.


We want to use both the stereo images since the illuminations conditions are similar to get a better result.

The most effective and efficient way is to set up tiles of 300x300 pixels with 100 pixels overlap to avoid artifacts (the so-called padding)
Smoothness weight is set to 0.7 and should avoid major artifacts and reduce the "streaks" typical of this technique.
Maximum iterations is set to 10.
We will use the bundle adjustment parameters as well.

On a core i7 quad core machine with 32 Gb of ram a 2000x3000 pixel image will take about 30 hours.

```
parallel_sfs -i run_adjust/spheroid-dem-trans-source-DEM.tif M1219322049LE.map.crop.cub M1219329084LE.map.crop.cub -o sfs_full_v_1/run --tile-size 300 --padding 100 --smoothness-weight 0.5 --initial-dem-constraint-weight 0.0001 --reflectance-type 1 --float-exposure --float-cameras --use-approx-camera-models --max-iterations 10 --use-approx-camera-models --use-rpc-approximation --crop-input-images --suppress-output --bundle-adjust-prefix run_ba/run
```

In a region with particular illuminations conditions such as completerly dark areas, shape from shading does not provide a solution. A shadow threshold must be set, using the "stereo_gui" mode of ASP. Here the two stere images are loaded and by selecting "threshold -> shadow threshold" and clickong multiple times on the two shaded images the thredhols values will appear in a dedicated window. T

```
stereo_gui M1219322049LE.map.crop_pit2.cub M1219329084LE.map.crop_pit2.cub run/out

```


Newer versions of ASP (2.7 or higher) have a slightly modified code. This is an example of what has been performed successfully on Marius Hills with shadow threshold enabled "--shadow-threshold" and the two values (one per image) and also model shadows taken into account with the "--model-shadows".
After 1 iteration of shape from shading the DEM is perfectly adherent to the original one.

```
parallel_sfs -i run-DEM.tif -n 1 -o prova_oct_2020_v10/run M1219322049LE.map.crop_pit2.cub M1219329084LE.map.crop_pit2.cub  --tile-size 100 --padding 50 --reflectance-type 1 --smoothness-weight 0.03 --initial-dem-constraint-weight 0.0001 --use-approx-camera-models --crop-input-images --suppress-output --bundle-adjust-prefix run_ba_pit2/run --shadow-thresholds "0.00205592322163283825 0.00235896115191280842" --model-shadows

```
