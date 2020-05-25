# LROC_NAC_stereo
This is to set up a workflow for generating decent quality LROC NAC stereo DEMs


crop from=M1219322049LE.map.cub to=M1219322049LE.map.crop.cub sample=2200 nsamples=5000 line=30000 nlines=5000

crop from=M1219329084LE.map.cub to=M1219329084LE.map.crop.cub sample=2200 nsamples=5000 line=30000 nlines=5000

bundle_adjust M1219322049LE.map.crop.cub M1219329084LE.map.crop.cub --min-matches 1 -o run_ba/run

stereo M1219322049LE.map.crop.cub M1219329084LE.map.crop.cub run_adjust/run --subpixel-mode 2 --bundle-adjust-prefix run_ba/run —threads 16

PC_Align needs to align the first obtained point cloud to a reference DEM. We chose the ~60 m esolution LOLA+Kaguya merged DEM. For the sake of this analysis we will crop it on a broader region

gdal_translate -projwin -1733458.7644640056 438880.4473252905 -1709916.8672144127 419142.4950550104 -of GTiff DEM_Kaguya.tif LOLA+Kaguya_crop.tif

point2dem -r moon run_full2/run-PC.tif --t_srs "+proj=eqc +lat_0=0 +lon_0=0 +a=1737400 +b=1737400 +units=m +no_defs" --nodata -32767 --dem-hole-fill-len 100 --orthoimage-hole-fill-len 100 --remove-outliers-params 75.0 3.0 --orthoimage run_full2/run-L.tif -o run_full2/ortho-L

