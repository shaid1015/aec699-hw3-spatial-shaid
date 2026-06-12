---
title: "Spatial analysis and regression"
author: 
  name: Arman Shaid
  affiliation: Oregon State University
date: "</br>11 June 2026"  # can also use: # </br>11 June 2026
output:
  html_document:
    theme: flatly  # you can experiment with other themes! for example, yeti
    highlight: haddock 
    toc: yes
    toc_depth: 3
    toc_float: yes
    keep_md: true
---



## Goal and requirements

The goal of this assignment is to practice spatial analysis and regression tools. 

Some additional points:  

- You can create a new GitHub repo for this assignment.  
- Name your GitHub repo something like `aec699-hw3-spatial-lastname`.  
- Make sure to organize your data into a `raw_data/` folder and your code into a `scripts/` or `code/` folder.  
- Do not forget to knit the assignment (click the “Knit” button, or press `Ctrl+Shift+K`) before submitting.  
- **Please put your (written) answers in bold.**  

### What you will be graded on

- Is the code correct?  
- Are the outcomes correct?  
- Did you show your work?  
- Are the figures clear?  
- Did you come up with creative empirical tests?  
- Did you follow the instructions of the assignment?  

## Preliminaries

### Load libraries

It is a good idea to load your libraries at the top of the Rmd document so that everyone can see what you are using. Similarly, it is good practice to set this type of chunk with `cache=FALSE` to ensure that the libraries are dynamically loaded each time you knit the document.

*Hint: I have only added the libraries needed to download and read the data. You will need to load additional libraries to complete this assignment. Add them here once you discover that you need them.* 


``` r
if (!require("pacman")) install.packages("pacman")  # install the pacman package if necessary
pacman::p_load(here,tidyverse,sf,terra,exactextractr,modelsummary)  # install other packages using pacman::p_load()
`%ni%` <- Negate(`%in%`)  # "not in" function
```

### Read in the data

The following chunks will set up the file structure for the assignment. Adjust to make sure it matches your working directory. 


``` r
# set up folders
if (!dir.exists(here("raw_data"))) dir.create(here("raw_data"))
if (!dir.exists(here("raw_data/census"))) dir.create(here("raw_data/census"))
if (!dir.exists(here("raw_data/prism"))) dir.create(here("raw_data/prism"))
if (!dir.exists(here("raw_data/prism/normal"))) dir.create(here("raw_data/prism/normal"))
if (!dir.exists(here("raw_data/prism/tmax"))) dir.create(here("raw_data/prism/tmax"))
```

This next chunk sets a .gitignore file for your repo. It ensures that raw data does not get uploaded to your GitHub repo. The first part checks if you already have a .gitignore file (which is a good idea). The second part creates a .gitignore file for you if not. You will know it is working if you do not have any raw data files in your "Git" pane in RStudio. 


``` r
if (file.exists(".gitignore")) {
  gitignore_contents <- read_lines(".gitignore")
  if ("raw_data/" %ni% gitignore_contents) {
    write_lines("raw_data/",".gitignore",append=T)
  }
} else {
  write_lines("raw_data/",".gitignore")
}
```

The next line reads in some county population data from the US Census. I have pre-cleaned it. You might consider moving it to your newly created `raw_data` folder. 


``` r
county_pop <- read_csv(here("raw_data/county_pop.csv"))
```

Next, we will read in a county shapefile from the US Census. 


``` r
download.file("https://www2.census.gov/geo/tiger/GENZ2020/shp/cb_2020_us_county_20m.zip",
              here("raw_data/census/cb_2020_us_county_20m.zip"))
unzip(here("raw_data/census/cb_2020_us_county_20m.zip"),exdir=here("raw_data/census"))
```

Lastly, we will use a few rasters from the [PRISM weather data](https://prism.oregonstate.edu), which is maintained at Oregon State University. This data is the most widely used climate data for United States-focused research, for example appearing in seminal climate economics papers like Schlenker and Roberts (2009) and Deschênes and Greenstone (2011). We will read in rasters for *climate normals*, which is the average temperature over 1990 to 2020, as well as *realized daily maximum temperature*. Both of these rasters are reported at a monthly level. We will pull information for January and July as representative "winter" and "summer" months. 


``` r
months <- str_pad(c(1,7),width=2,pad="0")
norm_file_src <- 
  paste0(
    "https://data.prism.oregonstate.edu/normals/us/4km/tmax/monthly/prism_tmax_us_25m_2020",
    months,
    "_avg_30y.zip"
  )
norm_file_dest <-
  paste0(
    here("raw_data/prism/normal/prism_tmax_us_25m_2020"),
    months,
    "_avg_30y.zip"
  )
tmax_file_src <-
  paste0(
    "https://data.prism.oregonstate.edu/time_series/us/an/4km/tmax/monthly/2025/prism_tmax_us_25m_2025",
    months,
    ".zip"
  )
tmax_file_dest <-
  paste0(
    here("raw_data/prism/tmax/prism_tmax_us_25m_2025"),
    months,
    ".zip"
  )
for (i in 1:length(norm_file_src)){
  download.file(norm_file_src[i], norm_file_dest[i])
  unzip(norm_file_dest[i],exdir=here("raw_data/prism/normal"))
  download.file(tmax_file_src[i], tmax_file_dest[i])
  unzip(tmax_file_dest[i],exdir=here("raw_data/prism/tmax"))
}
```

We are now ready to go.

## (1) Reading and cleaning our data

The main goal of the assignment is to practice spatial analysis and regression tools. The running example will relate climate to population growth in the contiguous United States. I am not saying any of these analyses are robust, but they are good practice for our data skills. 

### (1a) County shapefile

For the first part, read in the county shapefile. Print its output. 

*Hint: The relevant function is sf::read_sf().*


``` r
counties <- sf::read_sf(here("raw_data/census/cb_2020_us_county_20m.shp"))
counties
```

```
## Simple feature collection with 3221 features and 12 fields
## Geometry type: MULTIPOLYGON
## Dimension:     XY
## Bounding box:  xmin: -179.1743 ymin: 17.91377 xmax: 179.7739 ymax: 71.35256
## Geodetic CRS:  NAD83
## # A tibble: 3,221 × 13
##    STATEFP COUNTYFP COUNTYNS AFFGEOID     GEOID NAME  NAMELSAD STUSPS STATE_NAME
##    <chr>   <chr>    <chr>    <chr>        <chr> <chr> <chr>    <chr>  <chr>     
##  1 01      061      00161556 0500000US01… 01061 Gene… Geneva … AL     Alabama   
##  2 08      125      00198178 0500000US08… 08125 Yuma  Yuma Co… CO     Colorado  
##  3 17      177      01785076 0500000US17… 17177 Step… Stephen… IL     Illinois  
##  4 28      153      00695797 0500000US28… 28153 Wayne Wayne C… MS     Mississip…
##  5 34      041      00882237 0500000US34… 34041 Warr… Warren … NJ     New Jersey
##  6 46      051      01265782 0500000US46… 46051 Grant Grant C… SD     South Dak…
##  7 51      013      01480097 0500000US51… 51013 Arli… Arlingt… VA     Virginia  
##  8 21      007      00516850 0500000US21… 21007 Ball… Ballard… KY     Kentucky  
##  9 37      091      01026127 0500000US37… 37091 Hert… Hertfor… NC     North Car…
## 10 13      057      01685718 0500000US13… 13057 Cher… Cheroke… GA     Georgia   
## # ℹ 3,211 more rows
## # ℹ 4 more variables: LSAD <chr>, ALAND <dbl>, AWATER <dbl>,
## #   geometry <MULTIPOLYGON [°]>
```

**The county shapefile contains 3221 county-level polygons with attributes including `GEOID` (5-digit county FIPS), `STATEFP`, `NAME`, and the `geometry` column. The data are stored as an `sf` MULTIPOLYGON object.**

### (1b) Limiting to the contiguous United States

We will limit attention to the contiguous United States. Investigate if there are any extra states in your data and get rid of the non-contiguous states. (You can keep DC. It is contiguous.) Confirm that you have restricted attention to the contiguous United States with a simple ggplot. It does not need to be pretty. We are just exploring data at this stage.


``` r
# all STATEFP codes present in the shapefile
counties %>% st_drop_geometry() %>% distinct(STATEFP) %>% arrange(STATEFP)
```

```
## # A tibble: 52 × 1
##    STATEFP
##    <chr>  
##  1 01     
##  2 02     
##  3 04     
##  4 05     
##  5 06     
##  6 08     
##  7 09     
##  8 10     
##  9 11     
## 10 12     
## # ℹ 42 more rows
```


``` r
# non-contiguous STATEFP codes:
#   "02" Alaska, "15" Hawaii, "72" Puerto Rico,
#   "60" American Samoa, "66" Guam, "69" N. Mariana Is., "78" U.S. Virgin Is.
non_contig <- c("02","15","60","66","69","72","78")

counties <- counties %>% filter(STATEFP %ni% non_contig)
```


``` r
ggplot(counties) +
  geom_sf(fill = NA, color = "gray40", size = 0.1)
```

![](spatial_reg_files/figure-html/plot_contig-1.png)<!-- -->

**The raw shapefile included Alaska (`02`), Hawaii (`15`), Puerto Rico (`72`), and other non-contiguous territories. I dropped these `STATEFP` codes, keeping DC (`11`) since it is contiguous. The ggplot confirms the data are restricted to the lower 48 + DC.**

### (1c) Verifying information about the data

What coordinate reference system are the data in? How do you know?


``` r
st_crs(counties)
```

```
## Coordinate Reference System:
##   User input: NAD83 
##   wkt:
## GEOGCRS["NAD83",
##     DATUM["North American Datum 1983",
##         ELLIPSOID["GRS 1980",6378137,298.257222101,
##             LENGTHUNIT["metre",1]]],
##     PRIMEM["Greenwich",0,
##         ANGLEUNIT["degree",0.0174532925199433]],
##     CS[ellipsoidal,2],
##         AXIS["latitude",north,
##             ORDER[1],
##             ANGLEUNIT["degree",0.0174532925199433]],
##         AXIS["longitude",east,
##             ORDER[2],
##             ANGLEUNIT["degree",0.0174532925199433]],
##     ID["EPSG",4269]]
```

**The data are in NAD83 (EPSG:4269), a geographic coordinate system in degrees of latitude/longitude. I know this from two sources: (1) `st_crs(counties)` reports the CRS name and EPSG code directly, and (2) the shapefile bundle includes a `.prj` sidecar file (`cb_2020_us_county_20m.prj`) with the WKT definition, which `sf::read_sf()` reads automatically when loading the shapefile.**

### (1d) Reprojecting our data

Reproject the county data into a flat projection. I suggest US Albers Equal Area Conic ("EPSG:5070"). While you are at it, arrange the data by `GEOID`, which is the 5-digit county FIPS code. You should be able to pipe it so it is one command. Confirm the coordinate reference system with `st_crs()`.


``` r
counties <- counties %>%
  st_transform(crs = 5070) %>%
  arrange(GEOID)

st_crs(counties)
```

```
## Coordinate Reference System:
##   User input: EPSG:5070 
##   wkt:
## PROJCRS["NAD83 / Conus Albers",
##     BASEGEOGCRS["NAD83",
##         DATUM["North American Datum 1983",
##             ELLIPSOID["GRS 1980",6378137,298.257222101,
##                 LENGTHUNIT["metre",1]]],
##         PRIMEM["Greenwich",0,
##             ANGLEUNIT["degree",0.0174532925199433]],
##         ID["EPSG",4269]],
##     CONVERSION["Conus Albers",
##         METHOD["Albers Equal Area",
##             ID["EPSG",9822]],
##         PARAMETER["Latitude of false origin",23,
##             ANGLEUNIT["degree",0.0174532925199433],
##             ID["EPSG",8821]],
##         PARAMETER["Longitude of false origin",-96,
##             ANGLEUNIT["degree",0.0174532925199433],
##             ID["EPSG",8822]],
##         PARAMETER["Latitude of 1st standard parallel",29.5,
##             ANGLEUNIT["degree",0.0174532925199433],
##             ID["EPSG",8823]],
##         PARAMETER["Latitude of 2nd standard parallel",45.5,
##             ANGLEUNIT["degree",0.0174532925199433],
##             ID["EPSG",8824]],
##         PARAMETER["Easting at false origin",0,
##             LENGTHUNIT["metre",1],
##             ID["EPSG",8826]],
##         PARAMETER["Northing at false origin",0,
##             LENGTHUNIT["metre",1],
##             ID["EPSG",8827]]],
##     CS[Cartesian,2],
##         AXIS["easting (X)",east,
##             ORDER[1],
##             LENGTHUNIT["metre",1]],
##         AXIS["northing (Y)",north,
##             ORDER[2],
##             LENGTHUNIT["metre",1]],
##     USAGE[
##         SCOPE["Data analysis and small scale data presentation for contiguous lower 48 states."],
##         AREA["United States (USA) - CONUS onshore - Alabama; Arizona; Arkansas; California; Colorado; Connecticut; Delaware; Florida; Georgia; Idaho; Illinois; Indiana; Iowa; Kansas; Kentucky; Louisiana; Maine; Maryland; Massachusetts; Michigan; Minnesota; Mississippi; Missouri; Montana; Nebraska; Nevada; New Hampshire; New Jersey; New Mexico; New York; North Carolina; North Dakota; Ohio; Oklahoma; Oregon; Pennsylvania; Rhode Island; South Carolina; South Dakota; Tennessee; Texas; Utah; Vermont; Virginia; Washington; West Virginia; Wisconsin; Wyoming."],
##         BBOX[24.41,-124.79,49.38,-66.91]],
##     ID["EPSG",5070]]
```

**I reprojected the counties to US Albers Equal Area Conic (EPSG:5070) with `st_transform()` and sorted by `GEOID` in a single piped command. `st_crs()` confirms the new CRS is "NAD83 / Conus Albers" (EPSG:5070).**

### (1e) Matching climate data to our counties

Write a function that reads in the monthly climate normals and temperature data. You should compute zonal statistics for counties to report the average climate normal over 1990-2020 and the average temperature in 2025. Do not forget to put things in consistent coordinate reference systems. You should have 2 months per county. Print your dataframe when done.

*Hint: Use lapply() %>% bind_rows() or a purrr equivalent to create a county x month panel.*


``` r
get_climate_month <- function(mo) {
  norm_path <- here(
    "raw_data/prism/normal",
    paste0("prism_tmax_us_25m_2020", mo, "_avg_30y.tif")
  )
  tmax_path <- here(
    "raw_data/prism/tmax",
    paste0("prism_tmax_us_25m_2025", mo, ".tif")
  )

  norm_r <- terra::rast(norm_path)
  tmax_r <- terra::rast(tmax_path)

  # reproject counties to the raster CRS
  counties_r <- st_transform(counties, crs = st_crs(norm_r))

  tibble(
    GEOID       = counties_r$GEOID,
    month       = as.integer(mo),
    tmax_normal = exact_extract(norm_r, counties_r, "mean", progress = FALSE),
    tmax_2025   = exact_extract(tmax_r, counties_r, "mean", progress = FALSE)
  )
}
```


``` r
climate_df <- lapply(c("01","07"), get_climate_month) %>% bind_rows()
climate_df
```

```
## # A tibble: 6,216 × 4
##    GEOID month tmax_normal tmax_2025
##    <chr> <int>       <dbl>     <dbl>
##  1 01001     1        13.7     11.6 
##  2 01003     1        15.9     13.7 
##  3 01005     1        14.9     12.1 
##  4 01007     1        13.1     11.2 
##  5 01009     1        11.0      8.35
##  6 01011     1        14.4     11.8 
##  7 01013     1        15.0     12.3 
##  8 01015     1        11.7      9.78
##  9 01017     1        13.2     10.8 
## 10 01019     1        10.8      8.85
## # ℹ 6,206 more rows
```

**To build the county × month panel, I wrote a function get_climate_month() that takes a month string ("01" or "07") and does three things. First, it loads the matching PRISM rasters — the 1990–2020 climate normal and the 2025 monthly tmax. Second, it reprojects the county polygons into the raster's CRS so the two layers line up before any zonal computation. Third, it calls exactextractr::exact_extract() with "mean" to get the area-weighted average tmax inside each county, which is more accurate than nearest-cell extraction because it weights each grid cell by the fraction it overlaps with the polygon. I then ran the function over January and July and combined the outputs with bind_rows(). The result is a long panel with two rows per county — one for winter, one for summer — and columns for the GEOID, the month, the climate normal, and the 2025 realized tmax.**

### (1f) Spatial object

Join the zonal statistics to the county shapefile if you have not already. Make a conscious choice about whether to store your data in a wide or long format. Print the data and make sure the output shows the relevant data columns.


``` r
climate_wide <- climate_df %>%
  pivot_wider(
    id_cols     = GEOID,
    names_from  = month,
    values_from = c(tmax_normal, tmax_2025),
    names_glue  = "{.value}_{sprintf('%02d', month)}"
  )
climate_wide
```

```
## # A tibble: 3,108 × 5
##    GEOID tmax_normal_01 tmax_normal_07 tmax_2025_01 tmax_2025_07
##    <chr>          <dbl>          <dbl>        <dbl>        <dbl>
##  1 01001           13.7           32.7        11.6          33.8
##  2 01003           15.9           32.2        13.7          33.2
##  3 01005           14.9           32.9        12.1          33.7
##  4 01007           13.1           33.1        11.2          33.7
##  5 01009           11.0           31.7         8.35         31.8
##  6 01011           14.4           32.7        11.8          33.2
##  7 01013           15.0           32.9        12.3          33.2
##  8 01015           11.7           31.9         9.78         32.9
##  9 01017           13.2           32.4        10.8          32.8
## 10 01019           10.8           32.1         8.85         33.8
## # ℹ 3,098 more rows
```


``` r
counties <- counties %>% left_join(climate_wide, by = "GEOID")

counties %>%
  select(GEOID, NAME, STATEFP,
         tmax_normal_01, tmax_normal_07,
         tmax_2025_01,   tmax_2025_07,
         geometry)
```

```
## Simple feature collection with 3108 features and 7 fields
## Geometry type: MULTIPOLYGON
## Dimension:     XY
## Bounding box:  xmin: -2356114 ymin: 268660.9 xmax: 2258154 ymax: 3165722
## Projected CRS: NAD83 / Conus Albers
## # A tibble: 3,108 × 8
##    GEOID NAME    STATEFP tmax_normal_01 tmax_normal_07 tmax_2025_01 tmax_2025_07
##    <chr> <chr>   <chr>            <dbl>          <dbl>        <dbl>        <dbl>
##  1 01001 Autauga 01                13.7           32.7        11.6          33.8
##  2 01003 Baldwin 01                15.9           32.2        13.7          33.2
##  3 01005 Barbour 01                14.9           32.9        12.1          33.7
##  4 01007 Bibb    01                13.1           33.1        11.2          33.7
##  5 01009 Blount  01                11.0           31.7         8.35         31.8
##  6 01011 Bullock 01                14.4           32.7        11.8          33.2
##  7 01013 Butler  01                15.0           32.9        12.3          33.2
##  8 01015 Calhoun 01                11.7           31.9         9.78         32.9
##  9 01017 Chambe… 01                13.2           32.4        10.8          32.8
## 10 01019 Cherok… 01                10.8           32.1         8.85         33.8
## # ℹ 3,098 more rows
## # ℹ 1 more variable: geometry <MULTIPOLYGON [m]>
```

**I chose wide format. Each county now appears in a single row with four climate columns: tmax_normal_01, tmax_normal_07, tmax_2025_01, and tmax_2025_07. Wide format is convenient because every county has the same fixed set of climate measurements (two months × two variables), so a single row per county is the most compact and readable layout. I pivoted climate_df with pivot_wider() and joined it to the counties sf object by GEOID using left_join(), preserving the geometry column.**

## (2) Presenting your data

### (2a) January climate normal

Map the January climate normal from 1990 to 2020 at the county level. Make the map presentable. You can use nicer color scales than the default; consider `scale_fill_viridis_c()`, for example. You might also center the title using `theme()` commands. Or, you could get rid of the grid lines and axis text. It is up to you.

Which areas have the harshest winters?


``` r
ggplot(counties) +
  geom_sf(aes(fill = tmax_normal_01), color = NA) +
  scale_fill_viridis_c(option = "magma", name = "°C") +
  labs(
    title    = "January climate normal (1990–2020)",
    subtitle = "Average daily maximum temperature, contiguous U.S.",
    caption  = "Source: PRISM Climate Group"
  ) +
  theme_void() +
  theme(
    plot.title    = element_text(hjust = 0.5, face = "bold"),
    plot.subtitle = element_text(hjust = 0.5),
    plot.caption  = element_text(hjust = 0.5, size = 8),
    legend.position = "right"
  )
```

![](spatial_reg_files/figure-html/map_jan_normal-1.png)<!-- -->

**The harshest winters — the darkest counties on the map — are concentrated in the Upper Midwest and Northern Plains (North Dakota, Minnesota, northern Wisconsin and Michigan), the Northern Rockies (Montana, Wyoming, northern Idaho), and high-elevation parts of Colorado.**

### (2b) July climate normal

Map the July climate normal from 1990 to 2020. Make the map presentable. Which areas have the mildest summers?


``` r
ggplot(counties) +
  geom_sf(aes(fill = tmax_normal_07), color = NA) +
  scale_fill_viridis_c(option = "inferno", name = "°C") +
  labs(
    title    = "July climate normal (1990–2020)",
    subtitle = "Average daily maximum temperature, contiguous U.S.",
    caption  = "Source: PRISM Climate Group"
  ) +
  theme_void() +
  theme(
    plot.title    = element_text(hjust = 0.5, face = "bold"),
    plot.subtitle = element_text(hjust = 0.5),
    plot.caption  = element_text(hjust = 0.5, size = 8),
    legend.position = "right"
  )
```

![](spatial_reg_files/figure-html/map_jul_normal-1.png)<!-- -->

**The mildest summers — the darkest counties on the map — sit along the Pacific Northwest coast (coastal Washington, Oregon, and northern California), the high-elevation Rockies, and the upper tier of New England. **

### (2c) January 2025

Map the difference between January 2025 and the climate normal for 1990 to 2020. Which areas had a relatively warm January?


``` r
counties <- counties %>%
  mutate(jan_anom = tmax_2025_01 - tmax_normal_01)

ggplot(counties) +
  geom_sf(aes(fill = jan_anom), color = NA) +
  scale_fill_gradient2(
    low      = "steelblue",
    mid      = "white",
    high     = "firebrick",
    midpoint = 0,
    name     = "°C\nanomaly"
  ) +
  labs(
    title    = "January 2025 anomaly vs. 1990–2020 normal",
    subtitle = "Positive = warmer than normal",
    caption  = "Source: PRISM Climate Group"
  ) +
  theme_void() +
  theme(
    plot.title    = element_text(hjust = 0.5, face = "bold"),
    plot.subtitle = element_text(hjust = 0.5),
    plot.caption  = element_text(hjust = 0.5, size = 8),
    legend.position = "right"
  )
```

![](spatial_reg_files/figure-html/map_jan_anom-1.png)<!-- -->

**jan_anom = tmax_2025_01 - tmax_normal_01; positive values are counties warmer than their 1990–2020 January normal. A diverging red–white–blue scale centered at zero makes the sign easy to read. Relative to normal, January 2025 was warmest across the Northern Plains and Upper Midwest (the Dakotas, Minnesota, Wisconsin) and parts of the Northern Rockies, and coolest across the desert Southwest and southern Plains.**

### (2d) July 2025

Map the difference between July 2025 and the climate normal for 1990 to 2020. Which areas had a relatively warm July?


``` r
counties <- counties %>%
  mutate(jul_anom = tmax_2025_07 - tmax_normal_07)

ggplot(counties) +
  geom_sf(aes(fill = jul_anom), color = NA) +
  scale_fill_gradient2(
    low      = "steelblue",
    mid      = "white",
    high     = "firebrick",
    midpoint = 0,
    name     = "°C\nanomaly"
  ) +
  labs(
    title    = "July 2025 anomaly vs. 1990–2020 normal",
    subtitle = "Positive = warmer than normal",
    caption  = "Source: PRISM Climate Group"
  ) +
  theme_void() +
  theme(
    plot.title    = element_text(hjust = 0.5, face = "bold"),
    plot.subtitle = element_text(hjust = 0.5),
    plot.caption  = element_text(hjust = 0.5, size = 8),
    legend.position = "right"
  )
```

![](spatial_reg_files/figure-html/map_jul_anom-1.png)<!-- -->

**`jul_anom = tmax_2025_07 - tmax_normal_07`; positive values are counties warmer than their 1990–2020 July normal. July 2025 was warmer than normal across the Pacific Northwest (Oregon, Washington), the Southwest, and much of the eastern US. **

## (3) Relating population growth to climate

### (3a) Merging population

It is a good idea to create a non-spatial dataframe for the regression work we will do. Create a new dataframe (something like `county_df`) where you use `st_drop_geometry()` to get rid of the spatial elements from the county shapefile. Merge the population on. You can also consider dropping extraneous columns. Print your data.


``` r
county_df <- counties %>%
  st_drop_geometry() %>%
  left_join(county_pop %>% select(GEOID, pop_2000, pop_2020), by = "GEOID") %>%
  mutate(pop_pct_change = 100 * (pop_2020 - pop_2000) / pop_2000) %>%
  select(GEOID, NAME, STATEFP,
         pop_2000, pop_2020, pop_pct_change,
         tmax_normal_01, tmax_normal_07,
         tmax_2025_01,   tmax_2025_07,
         jan_anom, jul_anom)

county_df
```

```
## # A tibble: 3,108 × 12
##    GEOID NAME     STATEFP pop_2000 pop_2020 pop_pct_change tmax_normal_01
##    <chr> <chr>    <chr>      <dbl>    <dbl>          <dbl>          <dbl>
##  1 01001 Autauga  01         43671    58805          34.7            13.7
##  2 01003 Baldwin  01        140415   231767          65.1            15.9
##  3 01005 Barbour  01         29038    25223         -13.1            14.9
##  4 01007 Bibb     01         20826    22293           7.04           13.1
##  5 01009 Blount   01         51024    59134          15.9            11.0
##  6 01011 Bullock  01         11714    10357         -11.6            14.4
##  7 01013 Butler   01         21399    19051         -11.0            15.0
##  8 01015 Calhoun  01        112249   116441           3.73           11.7
##  9 01017 Chambers 01         36583    34772          -4.95           13.2
## 10 01019 Cherokee 01         23988    24971           4.10           10.8
## # ℹ 3,098 more rows
## # ℹ 5 more variables: tmax_normal_07 <dbl>, tmax_2025_01 <dbl>,
## #   tmax_2025_07 <dbl>, jan_anom <dbl>, jul_anom <dbl>
```

**I dropped the geometry with `st_drop_geometry()` to get a plain tibble, joined the 2000 and 2020 population counts from `county_pop` by `GEOID`, and computed `pop_pct_change` as the percent change between the two censuses — this will be the outcome in the regressions. I kept only the columns the regression work needs (identifiers, population, and the four climate variables plus the two anomalies) and dropped Census fields like `COUNTYFP`, `COUNTYNS`, `AFFGEOID`, `LSAD`, and `ALAND`/`AWATER`.**

### (3b) Regressing population change on climate

Run three regressions. First, regress the percent population change from 2000 to 2020 on January climate normal plus January climate normal squared. Next, do the same for July. Finally, include all of the variables (January and July) in a third regression. Output the results to a regression table. Can you conclude anything from these regressions?


``` r
m_jan <- lm(pop_pct_change ~ tmax_normal_01 + I(tmax_normal_01^2),
            data = county_df)
m_jul <- lm(pop_pct_change ~ tmax_normal_07 + I(tmax_normal_07^2),
            data = county_df)
m_both <- lm(pop_pct_change ~ tmax_normal_01 + I(tmax_normal_01^2) +
                              tmax_normal_07 + I(tmax_normal_07^2),
             data = county_df)

modelsummary(
  list("January only" = m_jan,
       "July only"    = m_jul,
       "Both"         = m_both),
  stars   = TRUE,
  gof_map = c("nobs", "r.squared", "adj.r.squared"),
  coef_rename = c(
    "tmax_normal_01"      = "January tmax normal",
    "I(tmax_normal_01^2)" = "January tmax normal²",
    "tmax_normal_07"      = "July tmax normal",
    "I(tmax_normal_07^2)" = "July tmax normal²"
  ),
  title = "Population % change (2000–2020) on climate normals"
)
```

```{=html}
<!-- preamble start -->

    <script src="https://cdn.jsdelivr.net/gh/vincentarelbundock/tinytable@main/inst/tinytable.js"></script>

    <script>
      // Create table-specific functions using external factory
      const tableFns_3y64pcapembq2yi9bnup = TinyTable.createTableFunctions("tinytable_3y64pcapembq2yi9bnup");
      // tinytable span after
      window.addEventListener('load', function () {
          var cellsToStyle = [
            // tinytable style arrays after
          { positions: [ { i: '13', j: 2 }, { i: '13', j: 3 }, { i: '13', j: 4 } ], css_id: 'tinytable_css_kklun795m6t11r0a7kd5',}, 
          { positions: [ { i: '10', j: 2 }, { i: '10', j: 3 }, { i: '10', j: 4 } ], css_id: 'tinytable_css_oc5t8qzud8ogt0rrvcj4',}, 
          { positions: [ { i: '1', j: 2 }, { i: '2', j: 2 }, { i: '3', j: 2 }, { i: '4', j: 2 }, { i: '5', j: 2 }, { i: '6', j: 2 }, { i: '7', j: 2 }, { i: '8', j: 2 }, { i: '9', j: 2 }, { i: '11', j: 2 }, { i: '12', j: 2 }, { i: '1', j: 3 }, { i: '2', j: 3 }, { i: '3', j: 3 }, { i: '4', j: 3 }, { i: '5', j: 3 }, { i: '6', j: 3 }, { i: '7', j: 3 }, { i: '8', j: 3 }, { i: '9', j: 3 }, { i: '11', j: 3 }, { i: '12', j: 3 }, { i: '1', j: 4 }, { i: '2', j: 4 }, { i: '3', j: 4 }, { i: '4', j: 4 }, { i: '5', j: 4 }, { i: '6', j: 4 }, { i: '7', j: 4 }, { i: '8', j: 4 }, { i: '9', j: 4 }, { i: '11', j: 4 }, { i: '12', j: 4 } ], css_id: 'tinytable_css_9o6yu13mf15k7rutbq6j',}, 
          { positions: [ { i: '0', j: 2 }, { i: '0', j: 3 }, { i: '0', j: 4 } ], css_id: 'tinytable_css_1e483c16yybrf8q10i1q',}, 
          { positions: [ { i: '13', j: 1 } ], css_id: 'tinytable_css_txk4yr1v6276jlvj82uk',}, 
          { positions: [ { i: '10', j: 1 } ], css_id: 'tinytable_css_k0yrsuh6r70dnyij5ryj',}, 
          { positions: [ { i: '1', j: 1 }, { i: '2', j: 1 }, { i: '3', j: 1 }, { i: '4', j: 1 }, { i: '5', j: 1 }, { i: '6', j: 1 }, { i: '7', j: 1 }, { i: '8', j: 1 }, { i: '9', j: 1 }, { i: '11', j: 1 }, { i: '12', j: 1 } ], css_id: 'tinytable_css_ubf4ho47ijtegf3unnrx',}, 
          { positions: [ { i: '0', j: 1 } ], css_id: 'tinytable_css_26nam1m8w2rwfsqojnnt',}, 
          ];

          // Loop over the arrays to style the cells
          cellsToStyle.forEach(function (group) {
              group.positions.forEach(function (cell) {
                  tableFns_3y64pcapembq2yi9bnup.styleCell(cell.i, cell.j, group.css_id);
              });
          });
      });
    </script>

    <link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/vincentarelbundock/tinytable@main/inst/tinytable.css">
    <style>
    /* tinytable css entries after */
    #tinytable_3y64pcapembq2yi9bnup td.tinytable_css_kklun795m6t11r0a7kd5, #tinytable_3y64pcapembq2yi9bnup th.tinytable_css_kklun795m6t11r0a7kd5 {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.08em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: center }
    #tinytable_3y64pcapembq2yi9bnup td.tinytable_css_oc5t8qzud8ogt0rrvcj4, #tinytable_3y64pcapembq2yi9bnup th.tinytable_css_oc5t8qzud8ogt0rrvcj4 {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: center }
    #tinytable_3y64pcapembq2yi9bnup td.tinytable_css_9o6yu13mf15k7rutbq6j, #tinytable_3y64pcapembq2yi9bnup th.tinytable_css_9o6yu13mf15k7rutbq6j { text-align: center }
    #tinytable_3y64pcapembq2yi9bnup td.tinytable_css_1e483c16yybrf8q10i1q, #tinytable_3y64pcapembq2yi9bnup th.tinytable_css_1e483c16yybrf8q10i1q {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 1; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.08em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: center }
    #tinytable_3y64pcapembq2yi9bnup td.tinytable_css_txk4yr1v6276jlvj82uk, #tinytable_3y64pcapembq2yi9bnup th.tinytable_css_txk4yr1v6276jlvj82uk {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.08em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: left }
    #tinytable_3y64pcapembq2yi9bnup td.tinytable_css_k0yrsuh6r70dnyij5ryj, #tinytable_3y64pcapembq2yi9bnup th.tinytable_css_k0yrsuh6r70dnyij5ryj {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: left }
    #tinytable_3y64pcapembq2yi9bnup td.tinytable_css_ubf4ho47ijtegf3unnrx, #tinytable_3y64pcapembq2yi9bnup th.tinytable_css_ubf4ho47ijtegf3unnrx { text-align: left }
    #tinytable_3y64pcapembq2yi9bnup td.tinytable_css_26nam1m8w2rwfsqojnnt, #tinytable_3y64pcapembq2yi9bnup th.tinytable_css_26nam1m8w2rwfsqojnnt {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 1; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.08em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: left }
    </style>
    <div class="container">
      <table class="tinytable" id="tinytable_3y64pcapembq2yi9bnup" style="width: auto; margin-left: auto; margin-right: auto;" data-quarto-disable-processing='true'>
        <caption>Population % change (2000–2020) on climate normals</caption>
        <thead>
              <tr>
                <th scope="col" data-row="0" data-col="1"> </th>
                <th scope="col" data-row="0" data-col="2">January only</th>
                <th scope="col" data-row="0" data-col="3">July only</th>
                <th scope="col" data-row="0" data-col="4">Both</th>
              </tr>
        </thead>
        <tfoot><tr><td colspan='4'>+ p < 0.1, * p < 0.05, ** p < 0.01, *** p < 0.001</td></tr></tfoot>
        <tbody>
                <tr>
                  <td data-row="1" data-col="1">(Intercept)</td>
                  <td data-row="1" data-col="2">3.478***</td>
                  <td data-row="1" data-col="3">76.185*</td>
                  <td data-row="1" data-col="4">-3.609</td>
                </tr>
                <tr>
                  <td data-row="2" data-col="1"></td>
                  <td data-row="2" data-col="2">(0.541)</td>
                  <td data-row="2" data-col="3">(31.659)</td>
                  <td data-row="2" data-col="4">(32.798)</td>
                </tr>
                <tr>
                  <td data-row="3" data-col="1">January tmax normal</td>
                  <td data-row="3" data-col="2">0.241*</td>
                  <td data-row="3" data-col="3"></td>
                  <td data-row="3" data-col="4">0.878***</td>
                </tr>
                <tr>
                  <td data-row="4" data-col="1"></td>
                  <td data-row="4" data-col="2">(0.110)</td>
                  <td data-row="4" data-col="3"></td>
                  <td data-row="4" data-col="4">(0.134)</td>
                </tr>
                <tr>
                  <td data-row="5" data-col="1">January tmax normal²</td>
                  <td data-row="5" data-col="2">0.023**</td>
                  <td data-row="5" data-col="3"></td>
                  <td data-row="5" data-col="4">0.024**</td>
                </tr>
                <tr>
                  <td data-row="6" data-col="1"></td>
                  <td data-row="6" data-col="2">(0.007)</td>
                  <td data-row="6" data-col="3"></td>
                  <td data-row="6" data-col="4">(0.008)</td>
                </tr>
                <tr>
                  <td data-row="7" data-col="1">July tmax normal</td>
                  <td data-row="7" data-col="2"></td>
                  <td data-row="7" data-col="3">-4.950*</td>
                  <td data-row="7" data-col="4">2.191</td>
                </tr>
                <tr>
                  <td data-row="8" data-col="1"></td>
                  <td data-row="8" data-col="2"></td>
                  <td data-row="8" data-col="3">(2.115)</td>
                  <td data-row="8" data-col="4">(2.215)</td>
                </tr>
                <tr>
                  <td data-row="9" data-col="1">July tmax normal²</td>
                  <td data-row="9" data-col="2"></td>
                  <td data-row="9" data-col="3">0.087*</td>
                  <td data-row="9" data-col="4">-0.068+</td>
                </tr>
                <tr>
                  <td data-row="10" data-col="1"></td>
                  <td data-row="10" data-col="2"></td>
                  <td data-row="10" data-col="3">(0.035)</td>
                  <td data-row="10" data-col="4">(0.038)</td>
                </tr>
                <tr>
                  <td data-row="11" data-col="1">Num.Obs.</td>
                  <td data-row="11" data-col="2">3106</td>
                  <td data-row="11" data-col="3">3106</td>
                  <td data-row="11" data-col="4">3106</td>
                </tr>
                <tr>
                  <td data-row="12" data-col="1">R2</td>
                  <td data-row="12" data-col="2">0.028</td>
                  <td data-row="12" data-col="3">0.003</td>
                  <td data-row="12" data-col="4">0.049</td>
                </tr>
                <tr>
                  <td data-row="13" data-col="1">R2 Adj.</td>
                  <td data-row="13" data-col="2">0.028</td>
                  <td data-row="13" data-col="3">0.002</td>
                  <td data-row="13" data-col="4">0.048</td>
                </tr>
        </tbody>
      </table>
    </div>
<!-- hack to avoid NA insertion in last line -->
```

**January and July climate normals are both statistically associated with population growth, but the relationships are non-linear. Warmer winters (January) are associated with faster growth in a convex pattern, while July shows a U-shaped relationship where moderate summer heat is most associated with slower growth. When combined, January dominates and July loses significance. R² values are low (~0.03–0.05), so climate explains little of the overall variation, and these results reflect correlation rather than causation.**

### (3c) Investigating "warming" and population growth

Consider an "ad hoc" measure of warming, which is the 2025 temperature minus the 1990-2020 normal. Does this measure correlate with population change for either January or July "warming"? Is this enough evidence to say anything about warming and population growth with this "ad hoc" measure? Use a regression table like in (3d) to provide an answer.


``` r
m_jan_w  <- lm(pop_pct_change ~ jan_anom, data = county_df)
m_jul_w  <- lm(pop_pct_change ~ jul_anom, data = county_df)
m_both_w <- lm(pop_pct_change ~ jan_anom + jul_anom, data = county_df)

modelsummary(
  list("January warming" = m_jan_w,
       "July warming"    = m_jul_w,
       "Both"            = m_both_w),
  stars   = TRUE,
  gof_map = c("nobs", "r.squared", "adj.r.squared"),
  coef_rename = c(
    "jan_anom" = "January warming (°C)",
    "jul_anom" = "July warming (°C)"
  ),
  title = "Population % change (2000–2020) on ad hoc warming"
)
```

```{=html}
<!-- preamble start -->

    <script src="https://cdn.jsdelivr.net/gh/vincentarelbundock/tinytable@main/inst/tinytable.js"></script>

    <script>
      // Create table-specific functions using external factory
      const tableFns_odnzyge12t90d8cg2p64 = TinyTable.createTableFunctions("tinytable_odnzyge12t90d8cg2p64");
      // tinytable span after
      window.addEventListener('load', function () {
          var cellsToStyle = [
            // tinytable style arrays after
          { positions: [ { i: '9', j: 2 }, { i: '9', j: 3 }, { i: '9', j: 4 } ], css_id: 'tinytable_css_0g8huv58e1z326anyixv',}, 
          { positions: [ { i: '6', j: 2 }, { i: '6', j: 3 }, { i: '6', j: 4 } ], css_id: 'tinytable_css_u6rcud1ki5gtjiwssbza',}, 
          { positions: [ { i: '1', j: 2 }, { i: '2', j: 2 }, { i: '3', j: 2 }, { i: '4', j: 2 }, { i: '5', j: 2 }, { i: '7', j: 2 }, { i: '8', j: 2 }, { i: '1', j: 3 }, { i: '2', j: 3 }, { i: '3', j: 3 }, { i: '4', j: 3 }, { i: '5', j: 3 }, { i: '7', j: 3 }, { i: '8', j: 3 }, { i: '1', j: 4 }, { i: '2', j: 4 }, { i: '3', j: 4 }, { i: '4', j: 4 }, { i: '5', j: 4 }, { i: '7', j: 4 }, { i: '8', j: 4 } ], css_id: 'tinytable_css_v8l21ix7k4pnu5rqfvef',}, 
          { positions: [ { i: '0', j: 2 }, { i: '0', j: 3 }, { i: '0', j: 4 } ], css_id: 'tinytable_css_0ynjvqtjjvjojvkal259',}, 
          { positions: [ { i: '9', j: 1 } ], css_id: 'tinytable_css_2uq1nx8rj0hlwumvdbgr',}, 
          { positions: [ { i: '6', j: 1 } ], css_id: 'tinytable_css_duzh6pklmkzjlu5il0ez',}, 
          { positions: [ { i: '1', j: 1 }, { i: '2', j: 1 }, { i: '3', j: 1 }, { i: '4', j: 1 }, { i: '5', j: 1 }, { i: '7', j: 1 }, { i: '8', j: 1 } ], css_id: 'tinytable_css_rys65ujx698lvhokkb5r',}, 
          { positions: [ { i: '0', j: 1 } ], css_id: 'tinytable_css_2h2r9zr9563j1vvp1y03',}, 
          ];

          // Loop over the arrays to style the cells
          cellsToStyle.forEach(function (group) {
              group.positions.forEach(function (cell) {
                  tableFns_odnzyge12t90d8cg2p64.styleCell(cell.i, cell.j, group.css_id);
              });
          });
      });
    </script>

    <link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/vincentarelbundock/tinytable@main/inst/tinytable.css">
    <style>
    /* tinytable css entries after */
    #tinytable_odnzyge12t90d8cg2p64 td.tinytable_css_0g8huv58e1z326anyixv, #tinytable_odnzyge12t90d8cg2p64 th.tinytable_css_0g8huv58e1z326anyixv {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.08em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: center }
    #tinytable_odnzyge12t90d8cg2p64 td.tinytable_css_u6rcud1ki5gtjiwssbza, #tinytable_odnzyge12t90d8cg2p64 th.tinytable_css_u6rcud1ki5gtjiwssbza {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: center }
    #tinytable_odnzyge12t90d8cg2p64 td.tinytable_css_v8l21ix7k4pnu5rqfvef, #tinytable_odnzyge12t90d8cg2p64 th.tinytable_css_v8l21ix7k4pnu5rqfvef { text-align: center }
    #tinytable_odnzyge12t90d8cg2p64 td.tinytable_css_0ynjvqtjjvjojvkal259, #tinytable_odnzyge12t90d8cg2p64 th.tinytable_css_0ynjvqtjjvjojvkal259 {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 1; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.08em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: center }
    #tinytable_odnzyge12t90d8cg2p64 td.tinytable_css_2uq1nx8rj0hlwumvdbgr, #tinytable_odnzyge12t90d8cg2p64 th.tinytable_css_2uq1nx8rj0hlwumvdbgr {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.08em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: left }
    #tinytable_odnzyge12t90d8cg2p64 td.tinytable_css_duzh6pklmkzjlu5il0ez, #tinytable_odnzyge12t90d8cg2p64 th.tinytable_css_duzh6pklmkzjlu5il0ez {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: left }
    #tinytable_odnzyge12t90d8cg2p64 td.tinytable_css_rys65ujx698lvhokkb5r, #tinytable_odnzyge12t90d8cg2p64 th.tinytable_css_rys65ujx698lvhokkb5r { text-align: left }
    #tinytable_odnzyge12t90d8cg2p64 td.tinytable_css_2h2r9zr9563j1vvp1y03, #tinytable_odnzyge12t90d8cg2p64 th.tinytable_css_2h2r9zr9563j1vvp1y03 {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 1; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.08em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: left }
    </style>
    <div class="container">
      <table class="tinytable" id="tinytable_odnzyge12t90d8cg2p64" style="width: auto; margin-left: auto; margin-right: auto;" data-quarto-disable-processing='true'>
        <caption>Population % change (2000–2020) on ad hoc warming</caption>
        <thead>
              <tr>
                <th scope="col" data-row="0" data-col="1"> </th>
                <th scope="col" data-row="0" data-col="2">January warming</th>
                <th scope="col" data-row="0" data-col="3">July warming</th>
                <th scope="col" data-row="0" data-col="4">Both</th>
              </tr>
        </thead>
        <tfoot><tr><td colspan='4'>+ p < 0.1, * p < 0.05, ** p < 0.01, *** p < 0.001</td></tr></tfoot>
        <tbody>
                <tr>
                  <td data-row="1" data-col="1">(Intercept)</td>
                  <td data-row="1" data-col="2">7.374***</td>
                  <td data-row="1" data-col="3">6.116***</td>
                  <td data-row="1" data-col="4">6.859***</td>
                </tr>
                <tr>
                  <td data-row="2" data-col="1"></td>
                  <td data-row="2" data-col="2">(0.742)</td>
                  <td data-row="2" data-col="3">(0.476)</td>
                  <td data-row="2" data-col="4">(0.759)</td>
                </tr>
                <tr>
                  <td data-row="3" data-col="1">January warming (°C)</td>
                  <td data-row="3" data-col="2">0.258</td>
                  <td data-row="3" data-col="3"></td>
                  <td data-row="3" data-col="4">0.379</td>
                </tr>
                <tr>
                  <td data-row="4" data-col="1"></td>
                  <td data-row="4" data-col="2">(0.300)</td>
                  <td data-row="4" data-col="3"></td>
                  <td data-row="4" data-col="4">(0.302)</td>
                </tr>
                <tr>
                  <td data-row="5" data-col="1">July warming (°C)</td>
                  <td data-row="5" data-col="2"></td>
                  <td data-row="5" data-col="3">1.168**</td>
                  <td data-row="5" data-col="4">1.231**</td>
                </tr>
                <tr>
                  <td data-row="6" data-col="1"></td>
                  <td data-row="6" data-col="2"></td>
                  <td data-row="6" data-col="3">(0.388)</td>
                  <td data-row="6" data-col="4">(0.392)</td>
                </tr>
                <tr>
                  <td data-row="7" data-col="1">Num.Obs.</td>
                  <td data-row="7" data-col="2">3106</td>
                  <td data-row="7" data-col="3">3106</td>
                  <td data-row="7" data-col="4">3106</td>
                </tr>
                <tr>
                  <td data-row="8" data-col="1">R2</td>
                  <td data-row="8" data-col="2">0.000</td>
                  <td data-row="8" data-col="3">0.003</td>
                  <td data-row="8" data-col="4">0.003</td>
                </tr>
                <tr>
                  <td data-row="9" data-col="1">R2 Adj.</td>
                  <td data-row="9" data-col="2">-0.000</td>
                  <td data-row="9" data-col="3">0.003</td>
                  <td data-row="9" data-col="4">0.003</td>
                </tr>
        </tbody>
      </table>
    </div>
<!-- hack to avoid NA insertion in last line -->
```

**The ad hoc warming measure (2025 tmax minus 1990–2020 normal) shows no significant association with population growth for January and a small positive association for July. However, this is not credible evidence about warming and population growth. The outcome (population change 2000–2020) was already realized before 2025, so the temporal ordering is backwards — 2025 temperatures cannot have caused earlier migration. A single year of anomalies also reflects weather noise rather than long-run climate change. This measure is not a valid test of whether warming affects population growth.**

### (3d) A more sophisticated approach

Come up with a more sophisticated econometric method than just regressing on temperature + temperature squared. Present the results in a figure or table.


``` r
# baseline (same as (3b) "Both") — pooled OLS, no controls
m_base <- lm(
  pop_pct_change ~ tmax_normal_01 + I(tmax_normal_01^2) +
                   tmax_normal_07 + I(tmax_normal_07^2),
  data = county_df
)

# add state fixed effects
m_fe <- lm(
  pop_pct_change ~ tmax_normal_01 + I(tmax_normal_01^2) +
                   tmax_normal_07 + I(tmax_normal_07^2) +
                   factor(STATEFP),
  data = county_df
)

# add state FE + initial population (mean-reversion control)
m_fe_pop <- lm(
  pop_pct_change ~ tmax_normal_01 + I(tmax_normal_01^2) +
                   tmax_normal_07 + I(tmax_normal_07^2) +
                   log(pop_2000) + factor(STATEFP),
  data = county_df
)

modelsummary(
  list("Pooled OLS"          = m_base,
       "+ State FE"          = m_fe,
       "+ State FE + log pop 2000" = m_fe_pop),
  stars     = TRUE,
  coef_omit = "factor\\(STATEFP\\)",
  gof_map   = c("nobs", "r.squared", "adj.r.squared"),
  coef_rename = c(
    "tmax_normal_01"      = "January tmax normal",
    "I(tmax_normal_01^2)" = "January tmax normal²",
    "tmax_normal_07"      = "July tmax normal",
    "I(tmax_normal_07^2)" = "July tmax normal²",
    "log(pop_2000)"       = "log(pop 2000)"
  ),
  add_rows = tibble::tribble(
    ~term,                ~`Pooled OLS`, ~`+ State FE`, ~`+ State FE + log pop 2000`,
    "State fixed effects", "No",          "Yes",         "Yes"
  ),
  title = "Population % change (2000–2020) — more sophisticated specifications"
)
```

```{=html}
<!-- preamble start -->

    <script src="https://cdn.jsdelivr.net/gh/vincentarelbundock/tinytable@main/inst/tinytable.js"></script>

    <script>
      // Create table-specific functions using external factory
      const tableFns_hj7pi21y6dulj4tc2nkj = TinyTable.createTableFunctions("tinytable_hj7pi21y6dulj4tc2nkj");
      // tinytable span after
      window.addEventListener('load', function () {
          var cellsToStyle = [
            // tinytable style arrays after
          { positions: [ { i: '16', j: 2 }, { i: '16', j: 3 }, { i: '16', j: 4 } ], css_id: 'tinytable_css_do6yyx8te55i6xi8n0s2',}, 
          { positions: [ { i: '12', j: 2 }, { i: '12', j: 3 }, { i: '12', j: 4 } ], css_id: 'tinytable_css_dpxgjqrr8yxdegkglxox',}, 
          { positions: [ { i: '1', j: 2 }, { i: '2', j: 2 }, { i: '3', j: 2 }, { i: '4', j: 2 }, { i: '5', j: 2 }, { i: '6', j: 2 }, { i: '7', j: 2 }, { i: '8', j: 2 }, { i: '9', j: 2 }, { i: '10', j: 2 }, { i: '11', j: 2 }, { i: '13', j: 2 }, { i: '14', j: 2 }, { i: '15', j: 2 }, { i: '1', j: 3 }, { i: '2', j: 3 }, { i: '3', j: 3 }, { i: '4', j: 3 }, { i: '5', j: 3 }, { i: '6', j: 3 }, { i: '7', j: 3 }, { i: '8', j: 3 }, { i: '9', j: 3 }, { i: '10', j: 3 }, { i: '11', j: 3 }, { i: '13', j: 3 }, { i: '14', j: 3 }, { i: '15', j: 3 }, { i: '1', j: 4 }, { i: '2', j: 4 }, { i: '3', j: 4 }, { i: '4', j: 4 }, { i: '5', j: 4 }, { i: '6', j: 4 }, { i: '7', j: 4 }, { i: '8', j: 4 }, { i: '9', j: 4 }, { i: '10', j: 4 }, { i: '11', j: 4 }, { i: '13', j: 4 }, { i: '14', j: 4 }, { i: '15', j: 4 } ], css_id: 'tinytable_css_tmqv9dfa2nrcebgmtkcx',}, 
          { positions: [ { i: '0', j: 2 }, { i: '0', j: 3 }, { i: '0', j: 4 } ], css_id: 'tinytable_css_m7ser5u3vw18eoolu2rh',}, 
          { positions: [ { i: '16', j: 1 } ], css_id: 'tinytable_css_u5imbc509ret24fw87pu',}, 
          { positions: [ { i: '12', j: 1 } ], css_id: 'tinytable_css_3kmxfx8uvviuu75mfqt3',}, 
          { positions: [ { i: '1', j: 1 }, { i: '2', j: 1 }, { i: '3', j: 1 }, { i: '4', j: 1 }, { i: '5', j: 1 }, { i: '6', j: 1 }, { i: '7', j: 1 }, { i: '8', j: 1 }, { i: '9', j: 1 }, { i: '10', j: 1 }, { i: '11', j: 1 }, { i: '13', j: 1 }, { i: '14', j: 1 }, { i: '15', j: 1 } ], css_id: 'tinytable_css_tvr59is8ff37625nbf5v',}, 
          { positions: [ { i: '0', j: 1 } ], css_id: 'tinytable_css_n9ldrxdcasvxt9nve0eu',}, 
          ];

          // Loop over the arrays to style the cells
          cellsToStyle.forEach(function (group) {
              group.positions.forEach(function (cell) {
                  tableFns_hj7pi21y6dulj4tc2nkj.styleCell(cell.i, cell.j, group.css_id);
              });
          });
      });
    </script>

    <link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/vincentarelbundock/tinytable@main/inst/tinytable.css">
    <style>
    /* tinytable css entries after */
    #tinytable_hj7pi21y6dulj4tc2nkj td.tinytable_css_do6yyx8te55i6xi8n0s2, #tinytable_hj7pi21y6dulj4tc2nkj th.tinytable_css_do6yyx8te55i6xi8n0s2 {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.08em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: center }
    #tinytable_hj7pi21y6dulj4tc2nkj td.tinytable_css_dpxgjqrr8yxdegkglxox, #tinytable_hj7pi21y6dulj4tc2nkj th.tinytable_css_dpxgjqrr8yxdegkglxox {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: center }
    #tinytable_hj7pi21y6dulj4tc2nkj td.tinytable_css_tmqv9dfa2nrcebgmtkcx, #tinytable_hj7pi21y6dulj4tc2nkj th.tinytable_css_tmqv9dfa2nrcebgmtkcx { text-align: center }
    #tinytable_hj7pi21y6dulj4tc2nkj td.tinytable_css_m7ser5u3vw18eoolu2rh, #tinytable_hj7pi21y6dulj4tc2nkj th.tinytable_css_m7ser5u3vw18eoolu2rh {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 1; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.08em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: center }
    #tinytable_hj7pi21y6dulj4tc2nkj td.tinytable_css_u5imbc509ret24fw87pu, #tinytable_hj7pi21y6dulj4tc2nkj th.tinytable_css_u5imbc509ret24fw87pu {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.08em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: left }
    #tinytable_hj7pi21y6dulj4tc2nkj td.tinytable_css_3kmxfx8uvviuu75mfqt3, #tinytable_hj7pi21y6dulj4tc2nkj th.tinytable_css_3kmxfx8uvviuu75mfqt3 {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 0; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.1em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: left }
    #tinytable_hj7pi21y6dulj4tc2nkj td.tinytable_css_tvr59is8ff37625nbf5v, #tinytable_hj7pi21y6dulj4tc2nkj th.tinytable_css_tvr59is8ff37625nbf5v { text-align: left }
    #tinytable_hj7pi21y6dulj4tc2nkj td.tinytable_css_n9ldrxdcasvxt9nve0eu, #tinytable_hj7pi21y6dulj4tc2nkj th.tinytable_css_n9ldrxdcasvxt9nve0eu {  position: relative; --border-bottom: 1; --border-left: 0; --border-right: 0; --border-top: 1; --line-color-bottom: var(--tt-line-color); --line-color-left: var(--tt-line-color); --line-color-right: var(--tt-line-color); --line-color-top: var(--tt-line-color); --line-width-bottom: 0.05em; --line-width-left: 0.1em; --line-width-right: 0.1em; --line-width-top: 0.08em; --trim-bottom-left: 0%; --trim-bottom-right: 0%; --trim-left-bottom: 0%; --trim-left-top: 0%; --trim-right-bottom: 0%; --trim-right-top: 0%; --trim-top-left: 0%; --trim-top-right: 0%; ; text-align: left }
    </style>
    <div class="container">
      <table class="tinytable" id="tinytable_hj7pi21y6dulj4tc2nkj" style="width: auto; margin-left: auto; margin-right: auto;" data-quarto-disable-processing='true'>
        <caption>Population % change (2000–2020) — more sophisticated specifications</caption>
        <thead>
              <tr>
                <th scope="col" data-row="0" data-col="1"> </th>
                <th scope="col" data-row="0" data-col="2">Pooled OLS</th>
                <th scope="col" data-row="0" data-col="3">+ State FE</th>
                <th scope="col" data-row="0" data-col="4">+ State FE + log pop 2000</th>
              </tr>
        </thead>
        <tfoot><tr><td colspan='4'>+ p < 0.1, * p < 0.05, ** p < 0.01, *** p < 0.001</td></tr></tfoot>
        <tbody>
                <tr>
                  <td data-row="1" data-col="1">(Intercept)</td>
                  <td data-row="1" data-col="2">-3.609</td>
                  <td data-row="1" data-col="3">-71.175+</td>
                  <td data-row="1" data-col="4">-30.312</td>
                </tr>
                <tr>
                  <td data-row="2" data-col="1"></td>
                  <td data-row="2" data-col="2">(32.798)</td>
                  <td data-row="2" data-col="3">(40.981)</td>
                  <td data-row="2" data-col="4">(36.245)</td>
                </tr>
                <tr>
                  <td data-row="3" data-col="1">January tmax normal</td>
                  <td data-row="3" data-col="2">0.878***</td>
                  <td data-row="3" data-col="3">0.552+</td>
                  <td data-row="3" data-col="4">-0.100</td>
                </tr>
                <tr>
                  <td data-row="4" data-col="1"></td>
                  <td data-row="4" data-col="2">(0.134)</td>
                  <td data-row="4" data-col="3">(0.317)</td>
                  <td data-row="4" data-col="4">(0.281)</td>
                </tr>
                <tr>
                  <td data-row="5" data-col="1">January tmax normal²</td>
                  <td data-row="5" data-col="2">0.024**</td>
                  <td data-row="5" data-col="3">-0.009</td>
                  <td data-row="5" data-col="4">-0.025*</td>
                </tr>
                <tr>
                  <td data-row="6" data-col="1"></td>
                  <td data-row="6" data-col="2">(0.008)</td>
                  <td data-row="6" data-col="3">(0.015)</td>
                  <td data-row="6" data-col="4">(0.013)</td>
                </tr>
                <tr>
                  <td data-row="7" data-col="1">July tmax normal</td>
                  <td data-row="7" data-col="2">2.191</td>
                  <td data-row="7" data-col="3">5.156+</td>
                  <td data-row="7" data-col="4">-3.592</td>
                </tr>
                <tr>
                  <td data-row="8" data-col="1"></td>
                  <td data-row="8" data-col="2">(2.215)</td>
                  <td data-row="8" data-col="3">(2.781)</td>
                  <td data-row="8" data-col="4">(2.475)</td>
                </tr>
                <tr>
                  <td data-row="9" data-col="1">July tmax normal²</td>
                  <td data-row="9" data-col="2">-0.068+</td>
                  <td data-row="9" data-col="3">-0.093+</td>
                  <td data-row="9" data-col="4">0.062</td>
                </tr>
                <tr>
                  <td data-row="10" data-col="1"></td>
                  <td data-row="10" data-col="2">(0.038)</td>
                  <td data-row="10" data-col="3">(0.048)</td>
                  <td data-row="10" data-col="4">(0.043)</td>
                </tr>
                <tr>
                  <td data-row="11" data-col="1">log(pop 2000)</td>
                  <td data-row="11" data-col="2"></td>
                  <td data-row="11" data-col="3"></td>
                  <td data-row="11" data-col="4">8.567***</td>
                </tr>
                <tr>
                  <td data-row="12" data-col="1"></td>
                  <td data-row="12" data-col="2"></td>
                  <td data-row="12" data-col="3"></td>
                  <td data-row="12" data-col="4">(0.293)</td>
                </tr>
                <tr>
                  <td data-row="13" data-col="1">Num.Obs.</td>
                  <td data-row="13" data-col="2">3106</td>
                  <td data-row="13" data-col="3">3106</td>
                  <td data-row="13" data-col="4">3106</td>
                </tr>
                <tr>
                  <td data-row="14" data-col="1">R2</td>
                  <td data-row="14" data-col="2">0.049</td>
                  <td data-row="14" data-col="3">0.162</td>
                  <td data-row="14" data-col="4">0.346</td>
                </tr>
                <tr>
                  <td data-row="15" data-col="1">R2 Adj.</td>
                  <td data-row="15" data-col="2">0.048</td>
                  <td data-row="15" data-col="3">0.148</td>
                  <td data-row="15" data-col="4">0.334</td>
                </tr>
                <tr>
                  <td data-row="16" data-col="1">State fixed effects</td>
                  <td data-row="16" data-col="2">No</td>
                  <td data-row="16" data-col="3">Yes</td>
                  <td data-row="16" data-col="4">Yes</td>
                </tr>
        </tbody>
      </table>
    </div>
<!-- hack to avoid NA insertion in last line -->
```

**
I present three progressive specifications: pooled OLS, OLS with state fixed effects, and OLS with state fixed effects plus log population in 2000. Compared to the simple regressions in (3b), adding state fixed effects substantially weakens the climate coefficients — January shrinks and July becomes fully insignificant. Once log population in 2000 is added, all climate terms lose significance except January tmax normal², which remains marginally significant and negative, suggesting a slight concavity in the January-growth relationship even after controls. Log population is strongly positive and dominant, and R² rises substantially across columns, showing that initial county size explains far more of population growth than climate. The contrast with (3b) is stark: the apparent climate-growth relationship was largely driven by state-level confounders and initial population size, not temperature itself.**

## AI Use Disclosure

I used Claude Code, a terminal-based AI assistant running directly inside RStudio, to help debug my code. This was genuinely new to me and I learned it through the workflow you showed us in the last class. Throughout the assignment, I understood what each chunk was doing and made judgment calls on the written interpretations myself, using the AI as a sounding board rather than a solution generator.
