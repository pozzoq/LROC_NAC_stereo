# LROC_NAC_stereo
This is to set up a workflow for generating decent quality LROC NAC stereo DEMs


crop from=M1219322049LE.map.cub to=M1219322049LE.map.crop.cub sample=2200 nsamples=5000 line=30000 nlines=5000

crop from=M1219329084LE.map.cub to=M1219329084LE.map.crop.cub sample=2200 nsamples=5000 line=30000 nlines=5000

bundle_adjust M1219322049LE.map.crop.cub M1219329084LE.map.crop.cub --min-matches 1 -o run_ba/run

stereo M1219322049LE.map.crop.cub M1219329084LE.map.crop.cub run_adjust/run --subpixel-mode 2 --bundle-adjust-prefix run_ba/run —threads 16
