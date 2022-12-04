#TO DO: ADJUST COLUMN NAMES
#TO DO: INCORPORATE SENSOR INTO PREDICTIONS
#TO DO: ADJUST POLYNOMIAL ISSUE

# Bird detectability offsets using QPAD v4

> This approach supsersedes the functionality of the QPAD package (but estimates are still stored in the package).

This repository contains geospatial layers that are consistent with QPAD v4 estimates.

## Installation

Install R then get the R dependencies:

```R
if (!requireNamespace("QPAD")) {
  if (!requireNamespace("remotes"))
    install.packages("remotes")
  remotes::install_github("psolymos/QPAD")
}
if (!requireNamespace("sp"))
  install.packages("sp")
if (!requireNamespace("maptools"))
  install.packages("maptools")
if (!requireNamespace("raster"))
  install.packages("raster")
if (!requireNamespace("intrval"))
  install.packages("intrval")
```

Download or clone [this GitHub repository](https://github.com/borealbirds/qpad-offsets):

```bash
git clone https://https://github.com/borealbirds/qpad-offsets.git
```

Then set your working directory to the project directory (`qpad-offsets`).
Easiest, if using RStudio, is to double click on the `offset.Rproj` file
which will open up the project and set the working directory.

## Usage

> An example can be found in the [`index.R`](index.R) file.

### Step 1. Load required packages and objects

```R
## load packages
library(QPAD)
library(maptools)
library(intrval)
library(raster)

## load v4 estimates
load_BAM_QPAD(version = 4)
if (!getBAMversion() %in% c("3", "4"))
  stop("This script requires BAM version 3 or version 4")

## read raster data
rlcc <- raster("./data/lcc.tif")
rtree <- raster("./data/tree.tif")
rtz <- raster("./data/utcoffset.tif")
rd1 <- raster("./data/seedgrow.tif")
crs <- proj4string(rtree)

## source functions
source("functions.R")
```

### Step 2. Define variables for your project

The date/time and coordinate specifications will make sure that required predictors are extracted in the way that match the estimates.

- the species ID need to be a single 4-letter AOU code (see `getBAMspecieslist()` for a full list)
- coordinates and time:  can be single values or vectors (shorter objects recycled)
  - `dt`: date, ISO 8601 in YYYY-MM-DD (0-padded)
  - `tm`: time, ISO 8601 in hh:mm (24 hr clock, 0-padded)
  - `lat`: latitude [WGS84 (EPSG: 4326)]
  - `lon`: longitude [WGS84 (EPSG: 4326)]
- methods descriptors: can be single value or vector (recycled as needed)
  - `dur`: duration in minutes
  - `dis`: distance in meters
- sensor type: currently either "PC" for human point counts or "ARU" for data that has been human-transcribed from autonomous recording units (ARUs)

```R
## species of interest
spp <- "OVEN"

## date and time
## https://en.wikipedia.org/wiki/ISO_8601
dt <- "2019-06-07" # ISO 8601 in YYYY-MM-DD (0-padded)
tm <- "05:20" # ISO 8601 in hh:mm (24 hr clock, 0-padded)

## spatial coordinates
lon <- -113.4938 # longitude WGS84 (EPSG: 4326)
lat <- 53.5461 # latitude WGS84 (EPSG: 4326)

## point count duration 
## and truncation distance (Inf for unlimited)
dur <- 10 # minutes
dis <- 100 # meters

## sensor
sensor <- "PC"
```

### Step 3. Organize predictors

This object can be reused for multiple species:

```R
x <- make_x(dt, tm, lon, lat, dur, dis, sensor)
str(x)
##'data.frame':	1 obs. of  8 variables:
## $ TSSR  : num 0.0089
## $ JDAY  : num 0.43
## $ DSLS  : num 0.14
## $ LCC2  : Factor w/ 2 levels "Forest","OpenWet": 2
## $ LCC4  : Factor w/ 4 levels "DecidMixed","Conif",..: 3
## $ TREE  : num 2.55
## $ MAXDUR: num 10
## $ MAXDIS: num 1
```

NOTE: CRS related warnings are due to [PROJ4 vs PROJ6](https://stackoverflow.com/questions/63727886/proj4-to-proj6-upgrade-and-discarded-datum-warnings) discrepancies when using GDAL > 3 because the `+datum=` part is deprecated.

### Step 4. Calculate offsets

`A` is the known or estimated area of survey, `p` is availability given presence, `q` is detectability given availability.

```R
o <- make_off(spp, x)
str(o)
##'data.frame':	1 obs. of  5 variables:
## $ p         : num 0.991
## $ q         : num 0.562
## $ A         : num 3.14
## $ correction: num 1.75
## $ offset    : num 0.559
```

NOTE: `offset` is `log(correction)`, `correction` = `A*p*q`, thus `offset=log(A) + log(p) + log(q)`.

Use a loop over multiple species:

```R
SPP <- getBAMspecieslist()
OFF <- matrix(0, nrow(x), length(SPP))
rownames(OFF) <- rownames(x) # your survey IDs here
colnames(OFF) <- SPP

for (spp in SPP) {
  cat(spp, "\n")
  flush.console()
  o <- make_off(spp, x)
  OFF[,spp] <- o$offset
}
str(OFF)
##num [1, 1:151] 0.1365 0.9699 0.0643 0.5917 -0.3132 ...
## - attr(*, "dimnames")=List of 2
##  ..$ : chr "17"
##  ..$ : chr [1:151] "ALFL" "AMCR" "AMGO" "AMPI" ...
```

*This repository originates from the archived <https://github.com/ABbiodiversity/recurring> project.*
