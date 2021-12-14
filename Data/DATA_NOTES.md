# Original Data
"CMS1 Data through 2019.xlsx" contains data logger data collected by FOCB
using a YSI EXO sonde and a seperate pCO2 sensor.

Column Name     | Contents                               | Units                         
----------------|----------------------------------------|------
Date	          | Dates, in a mixture of Excel Dates, Times, and Character Strings |
Time	          | Time of Day, similarly, in a mixture of formats  |
Water Depth     | Water depth (from pressure transducer on sonde)  | m
Temperature     | Temperature                          |   	C
Salinity	       | Salinity based on conductivity, roughly in PPT | PSU
DO	             | Dissolved oxygen                     | mg/l
DO%             | Percent oxygen saturation            |	%
pH	NBS          | pH, measured with a pH electrode, this on NBS scale | Unitless
Chl             | Chlorophyll-A (from florescence)     |	ug/l
pCO2            | Partial pressure of CO~2~            |	ppm
Month	          | Month                                | as integer 1 through 12   
Year	          | Year,                                | Four digit integer
Day             | Day of the month                     |	Integer
Hour	          | Time of day, to nearest hour         | 0 to 23
TA	             | Total alkalinity  (Calculated from pH and pCO~2~)  | uM/kg
DIC             | Dissolved Inorganic carbon (Calculated)    |	uM/kg
Omega Aragonite |	Solubility quotient aragonite (Calculated  | Unitless

# Derived Data
"FOCB Monitoring Sites SHORT NAMES.xlsx" is a hand edited version of data
received directly from Friends of Casco Bay.   The only
change is the addition of a column containing shorter names for FOCB
sampling locations, for use in SoCB graphics.

Column Name     | Contents                                      
----------------|-----------------------------------------------
Station_ID      | Friends of Casco Bay alphanumeric Side Code
Station_Name    | Longer Text name of each site
Alt_Name        | Shorter site name, convenient for maps and graphics
Town            | Town in which the sampling location is located
Y               | Latitude, assumed here to be WGS 1984
X               | Longitude, assumed here to be WGS 1984
Category        | "Surface" site or "Profile" site. 

Vertical profile data is available from profile sites, which are generally
offshore sites visited by boat.

"station_summary.csv" contains medians and means of recent FOCB water quality
data, by Station.  "Recent" here refers to the five year period 2015 through
2019. The data were derived from the raw data using simpel R code (not provided
here).
station
secchi_2_mn
secchi_2_med
sqrt_secchi_mn
sqrt_secchi_med
temperature_mn
temperature_med
salinity_mn
salinity_med
do_mn
do_med
pctsat_mn
pctsat_med
pH_mn
pH_med
chl_mn
chl_med
log_chl_mn
log_chl_med
log1_chl_mn
log1_chl_med

## Monitoring Locations
The shapefile 'monitoring_locations' was derived from the Excel spreadsheet
"FOCB Monitoring Sites.xlsx".  This file does not include geographic information
for the "CMS3" station, which FOCB began using in 2020.

## Near Impervious Cover Estimates
The file "focb_monitoring_imperv.csv" contains estimates of the relative percent 
cover of impervious area within 100 meters, 500 meters, and 1000 meters of 
FOCB sampling locations.  We assembled this data as a potential predictor of
water quality conditions at sampling locations.

Impervious cover estimates (calculated only for station locations) were
based on Maine IF&W one meter pixel impervious cover data, which is based
largely on data from 2007. CBEP has a version of this impervious cover data for
the Casco Bay watershed towns in our GIS data archives. Analysis followed the
following steps. 

### Pepration of the data by GIS
1. Town by town IC data in a Data Catalog were assembled into a large `tif` 
   file using the "Mosaic Raster Catalog" item from the context menu from the
   ArcGIS table of contents.

2. We created a convex hull enclosing all of the FOCB Station locations
   using the "minimum bounding geometry" tool.  We buffered that polygon by 2000 
   meters.  The resulting polygon omitted several portions of the Bay likely to
   be important for other analyses, so we created an edited version by hand that
   included other portions of the Bay likely to be of interest (chiefly around
   several islands and the lower New Meadows).  While this area was added by 
   hand, we tried to ensure that it included all islands and an area 2000 m 
   landward of likely monitoring locations.  That polygon was saved as the 
   shapefile "cb_poly_buf".

3.  We made a tiff copy of the  data using the context menu item
   
   Data -> Export Data
   
   We clipped that copy of the data layer with the "Extract by Mask" 
   tool, and saved the results.

4. We used "Aggregate" to reduce the resolution of the impervious cover raster
   to a 5 meter resolution, summing the total impervious cover within each
   5m x 5m area, generating a raster with values from zero to 25. This
   speeds later processing, for a negligible reduction in precision.

5. We used "Focal Statistics" to generate rasters that show the cumulative area
   of impervious cover (in meters) within 100 m, 500 m, and 1000 m of each FOCB 
   sampling location. (The 1000m version was very slow to calculate.)

6. We generated a land cover raster layer based on the state's `cnty24p` data.
   We clipped that data layer to the mask polygon, merged all polygons to a
   single multipolygon and added a dummy attribute with a value of one.  We  
   convertedthat polygon layer to a TIFF raster with a 5 meter pixel using the 
   "Polygon to Raster (Conversion)" tool.  Every pixel has a value of one, but 
   an area of 25 square meters, so we need to  account for that later.

7. We used "Focal Statistics" to generate rasters that show the cumulative sum
   (NOT area) of the land cover raster within 100 m, 500 m, and 1000 m of each
   pixel.  (To get true land area, we still need to multiply values by
   25).
   
8. We extracted the values of the three rasters produced in step 5 and three
   rasters produced in step 7 at each Station location. We used  'Extract 
   Multi Values to Points'. (variable names are imperv_[radius] and 
   land[_radius] respectively).  
   
   We replaced any null values (-9999) with zeros, to account for points that
   lie more than the specified distance from land or impervious cover (using the
   field calculator).

9. We calculated (two versions of) percent cover with the Field Calculator.   
   *   We divided the total impervious cover within the specified distance by  
       the area of the circle sampled under step (5) ($\pi \cdot r^2$).  
   *   We divided the total impervious cover within the specified distance by  
       the estimated land area within each circle, for a percent impervious per 
       unit land. (Land area is 25 times the extracted value from the raster).  
   *   Variable names are pct_[radius] and pct_l_[radius], respectively for 
       percent based on total area and land area.  

10.  Impervious cover data was exported to the text file 
     "focb_monitoring_imperv.csv".
     
### Data Contents
Column Name     | Contents                                      
----------------|-----------------------------------------------
Station_ID      | Friends of Casco Bay alphanumeric Side Code
Station_Name    | Longer Text name of each site
Alt_Name        | Shorter site name, convenient for maps and graphics
Town            | Town in which the sampling location is located
Y               | Latitude, assumed here to be WGS 1984
X               | Longitude, assumed here to be WGS 1984
Category        | "Surface" site or "Profile" site. 
land_1000       | Total Land Area within 1000 meters of sampling location
imperv_100      | Impervious cover within 1000 meters (ArcGIS truncated the attribute name)
land_500        | Total Land Area within 500 meters of sampling location
imperv_500      | Impervious cover within 500 meters
land_100        | Total Land Area within 100 meters of sampling location
imperv_101      | Impervious cover within 100 meters (ArcGIS mangled teh attribute name to avoid a name conflict)
pct_100         | Percent of area within 100 m that is impervious cover
pct_500         | Percent of area within 500 m that is impervious cover
pct_1000        | Percent of area within 1000 m that is impervious cover
pct_l_1000      | Percent of LAND area within 1000 m that is impervious cover
pct_l_500       | Percent of LAND area within 500 m that is impervious cover
pct_l_100       | Percent of LAND area within 100 m that is impervious cover
     
