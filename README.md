# LROC_NAC_stereo
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
M1219322049LE.lronaccal.lronacecho.noproj.mosaic.norm.cub M1219329084LE.lronaccal.lronacecho.noproj.mosaic.norm.cub
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
stereo M1219322049LE.map.crop.cub M1219329084LE.map.crop.cub run_adjust/run --subpixel-mode 2 --bundle-adjust-prefix run_ba/run —threads 16
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

First we need to ensure that if we are running python 3 on our machine we modify 

