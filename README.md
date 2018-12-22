# Forest@risk 

*Marco Girardello* (marco.girardello@gmail.com) 

Python code developed as part of the Forest@risk project. This code calculates zonal statistics for three forest disturbance polygons datasets with hundreds of environmental layers.

```python
##################################################################################
# Extract zonal statistics for raster layers as part of the project forest@risk
#################################################################################

# import required modules
import numpy as np
import geopandas as gpd
import rasterio as rst
from rasterstats import zonal_stats
import glob
import os
import pandas as pd
import re

#----EFFIS: observed polygons

# read in polygon data
results_final = []
for filep in glob.glob('/ESS_Datasets/USERS/Marco/datasets/polygonsGRASS/EFFIS/*.shp'):
    # read in polygon
    tmpp = gpd.read_file(filep)
    # append polygon to list
    results_final.append(tmpp)

# ---- PFTs

# Loop through raster files
for filer in glob.glob('/ESS_Datasets/USERS/Marco/rasters/EU/pfts/*.tif'):
        # open raster with rasterio
        tmpr = rst.open(filer)
        # convert into array
        tmpar = tmpr.read(1)
        # insert 0s for no data values
        tmpar[tmpar == tmpr.nodata] = 0
        # get transformation parameter (needed for zonal stats)
        affine = tmpr.transform
        # get out names for variable of interest
        varname = os.path.basename(filer).split('.')[0]
        # Loop through vector files
        for filep in results_final:
            # zonal statisticfileslegs (sum)
            stats = zonal_stats(filep, tmpar, affine=affine, stats=['sum'], all_touched=True)
            # zonal statistics (count)
            totpixels = zonal_stats(filep, tmpar, affine=affine, stats=['count'], all_touched=True)
            # convert dictionaries into lists
            stats1 = [val for dic in stats for val in dic.values()]
            totpixels1 = [val for dic in totpixels for val in dic.values()]
            # store in pandas dataframe
            filep[varname] = stats1
            # total number of pixels
            varname1 = varname + '_pixels'
            filep[varname1] = totpixels1


# ----Static maps:

# Loop through raster files
for filer in glob.glob('/ESS_Datasets/USERS/Marco/rasters/EU/static1/*.tif'):
    # open raster with rasterio
    tmpr = rst.open(filer)
    # convert into array
    tmpar = tmpr.read(1)
    # if integer convert into floating point
    if ('float' in tmpar.dtype.type.__name__)==False:
        # set array values to floating
        tmpar = tmpar.astype('float')
    # insert 0s for no data values
    tmpar[tmpar == tmpr.nodata] = np.nan
    # get transformation parameter (needed for zonal stats)
    affine = tmpr.transform
    # get out names for variable of interest
    varname = os.path.basename(filer).split('.')[0]
    # Loop through vector files
    for filep in results_final:
        # zonal statistics (mean)
        stats = zonal_stats(filep, tmpar, affine=affine, stats=['mean'], all_touched=True)
        # convert dictionaries into lists
        stats1 = [val for dic in stats for val in dic.values()]
        # store in pandas dataframe
        filep[varname] = stats1


# ----Dynamic maps: event scale

# Loop through raster files
for filer in glob.glob('/ESS_Datasets/USERS/Marco/rasters/EU/dynamic1/*.tif'):
    # open raster with rasterio
    tmpr = rst.open(filer)
    # convert into array
    tmpar = tmpr.read(1)
    # if integer convert into floating point
    if ('float' in tmpar.dtype.type.__name__)==False:
        # set array values to floating
        tmpar = tmpar.astype('float')
    # insert 0s for nresults1o data values
    tmpar[tmpar == tmpr.nodata] = np.nan
    # get transformation parameter (needed for zonal stats)
    affine = tmpr.transform
    # exception for Aridity Index (lack consistent naming)
    if 'AI' in os.path.basename(filer):
        varname = os.path.basename(filer).split('.')[0].split('_')[0]
        year = os.path.basename(filer).split('.')[0].split('_')[1]
    # contains population name
    elif 'pop' in os.path.basename(filer):
           varname = os.path.basename(filer).split('.')[0].split('2')[0]
           numbs = re.findall('\d+', os.path.basename(filer).split('.')[0])
           # extract the number with the longest number of digits
           year = max(numbs, key=len)
    elif 'suppressionp' in os.path.basename(filer) or 'ignitionp' in os.path.basename(filer):
               varname = os.path.basename(filer).split('.')[0].split('_')[0]
               numbs = re.findall('\d+', os.path.basename(filer).split('.')[0])
               # extract the number with the longest number of digits
               year = max(numbs, key=len)
    else:
        # get out names for variable of interest
        varnametmp = os.path.basename(filer).split('.')[0]
        varname = varnametmp.split('_')[0] + "_" + varnametmp.split('_')[1] + "_" + varnametmp.split('_')[3]
        # get out name for variable of interest
        # parse numbers (could be year, resolution or season!)
        numbs = re.findall('\d+', os.path.basename(filer).split('.')[0])
        # extract the number with the longest number of digits
        year = max(numbs, key=len)
        # if year = 1999 skip iteration
    if year == '1999':
       pass
    else:
        # match year with polygon
        for filep in results_final:
            # create year filter
            if (str(filep.yearssn.unique()[0]) == year):
                # zonal statistics (mean)
                stats = zonal_stats(filep, tmpar, affine=affine, stats=['mean'], all_touched=True)
                # convert dictionaries into lists
                stats1 = [val for dic in stats for val in dic.values()]
                # store in pandas dataframe
                filep[varname] = stats1
            else:
                pass



# ----Dynamic maps: legacy effects

# exclude population density and fire suppression and ignition probabilities
filesleg = [fn for fn in glob.glob('/ESS_Datasets/USERS/Marco/rasters/EU/dynamic1/*.tif') if
 'pop' not in os.path.basename(fn) and 'suppressionp' not in os.path.basename(fn) and 'ignitionp' not in os.path.basename(fn)]

# Loop through raster files
for filer in filesleg:
    # open raster with rasterio
    tmpr = rst.open(filer)
    # exception for Aridity Index (lack consistent naming)
    if 'AI' in os.path.basename(filer):
        varname = os.path.basename(filer).split('.')[0].split('_')[0]
        year = os.path.basename(filer).split('.')[0].split('_')[1]
    else:
        # get out names for variable of interest
        varnametmp = os.path.basename(filer).split('.')[0]
        varname = varnametmp.split('_')[0] + "_" + varnametmp.split('_')[1] + "_" + varnametmp.split('_')[3]
        # parse numbers (could be year, resolution or season!)
        numbs = re.findall('\d+', os.path.basename(filer).split('.')[0])
        # extract the number with the longest number of digits
        year = max(numbs, key=len)
        if year=='1999':
           pass
        else:
            # extract year-1 from name
            year_m1 = str(int(year)-1)
            # path for raster year -1
            fpathym1 = '/ESS_Datasets/USERS/Marco/rasters/EU/dynamic1/' + str.replace(varnametmp, year, year_m1) + '.tif'
            # open raster with rasterio
            tmpr = rst.open(filer)
            # convert into array
            tmpar = tmpr.read(1)
            # if integer convert into floating point
            if ('float' in tmpar.dtype.type.__name__) == False:
               # set array values to floating
               tmpar = tmpar.astype('float')
            # get transformation parameter (needed for zonal stats)
            affine = tmpr.transform
            # match year with polygon
            for filep in results_final:
                # create year filter
                if (str(filep.yearssn.unique()[0]) == year):
                    # zonal statistics (mean)
                    stats = zonal_stats(filep, tmpar, affine=affine, stats=['mean'], all_touched=True)
                    # convert dictionaries into lists
                    stats1 = [val for dic in stats for val in dic.values()]
                    # store in pandas dataframe
                    filep[varname+'_ym1'] = stats1


# ----EFI tree cover maps:

# Loop through raster files

for filer in glob.glob('/ESS_Datasets/USERS/Marco/rasters/EU/EFItrees/*.tif'):
        # open raster with rasterio
        tmpr = rst.open(filer)
        # convert into array
        tmpar = tmpr.read(1)
        # insert 0s for no data values
        tmpar[tmpar == tmpr.nodata] = 0
        # get transformation parameter (needed for zonal stats)
        affine = tmpr.transform
        # get out names for variable of interest
        varname = os.path.basename(filer).split('.')[0]
        # Loop through vector files
        for filep in results_final:
            # zonal statistics (mean)
            stats = zonal_stats(filep, tmpar, affine=affine, stats=['sum'], all_touched=True)
            # convert dictionaries into lists
            stats1 = [val for dic in stats for val in dic.values()]
            # store in pandas dataframe
            filep[varname] = stats1


# concatenate list of pandas dataframes
results_final1 = pd.concat(results_final)
# delete geometry column
results_final1 = results_final1.drop(['geometry','height1'], axis=1)

# write results as csv file
results_final1.to_csv('/ESS_Datasets/USERS/Marco/rasters/EFFISObservedv2.csv', index=False)

```
