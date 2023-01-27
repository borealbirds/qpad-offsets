# Bird detectability offsets using QPAD v4

> This approach supsersedes the functionality of the QPAD package (but estimates are still stored in the package).

This repository contains geospatial layers that are consistent with QPAD v4 (and v3) estimates.

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

### Step 2. Create a dataframe of variables for your project

The date/time and coordinate specifications will make sure that required predictors are extracted in the way that match the estimates.

- time descriptors:
  - `dt`: date, YYYY-MM-DD (0-padded) https://en.wikipedia.org/wiki/ISO_8601
  - `tm`: time, hh:mm (24 hr clock, 0-padded) ## https://en.wikipedia.org/wiki/ISO_8601
- coordinates:
  - `lat`: latitude [WGS84 (EPSG: 4326)]
  - `lon`: longitude [WGS84 (EPSG: 4326)]
- methods descriptors:
  - `dur`: duration in minutes
  - `dis`: truncation distance in meters (Inf for unlimited)
  - `tagmeth`: either "PC" for human point counts, "1SPT" for autonomous recording unit (ARU) data that has been human-transcribed to the first detection per individual, or "1SPM" for ARU data that has been human-transcribed to the first detection per indvidual per minute. See https://www.wildtrax.ca/home/resources/guide/acoustic-data/acoustic-tagging-methods.html for details on ARU tagging methods.
```R
## dataframe
dat <- data.frame(date = c("2019-06-07", "2019-06-17", "2019-06-27"),
                  time = rep("05:20", 3), 
                  lon = rep(-115, 3),
                  lat = rep(53, 3),
                  dur = rep(10, 3), 
                  dist = rep(100, 3),
                  tagmeth = rep("PC", 3)) 
```

### Step 3. Organize predictors

Use the `tz` argument to indicate whether the times in your dataframe are local times or UTC.

The output object can be reused for multiple species (see Step 4).

```R
## timezone argument
tz <- "local"

## organize predictors
x <- make_x(dat, tz)
str(x)
# 'data.frame':	3 obs. of  9 variables:
#  $ TSSR  : num  0.0024 0.0044 0.0026
#  $ JDAY  : num  0.43 0.458 0.485
#  $ DSLS  : num  0.11 0.137 0.164
#  $ LCC2  : Factor w/ 2 levels "Forest","OpenWet": 2 2 2
#  $ LCC4  : Factor w/ 4 levels "DecidMixed","Conif",..: 3 3 3
#  $ TREE  : num  0.3 0.3 0.3
#  $ MAXDUR: num  10 10 10
#  $ MAXDIS: num  1 1 1
#  $ TM    : chr  "PC" "PC" "PC"
```

NOTE: CRS related warnings are due to [PROJ4 vs PROJ6](https://stackoverflow.com/questions/63727886/proj4-to-proj6-upgrade-and-discarded-datum-warnings) discrepancies when using GDAL > 3 because the `+datum=` part is deprecated.

### Step 4 Option 1. Calculate offsets

Use the `spp` argument to specify the species of interest. The species ID needs to be a single 4-letter AOU code (see `getBAMspecieslist()` for a full list).

Use the `useMeth` argument to indicate whether you would like offsets that take method (human point count, ARU with transcription method) into account: c("y", "n").

```R
## species of interest
spp <- "OVEN"

## take method into acount
useMeth <- "y"

o <- make_off(spp, x, useMeth)
str(o)
# 'data.frame':	3 obs. of  5 variables:
#  $ p         : num  0.971 0.961 0.949
#  $ q         : num  0.58 0.58 0.58
#  $ A         : num  3.14 3.14 3.14
#  $ correction: num  1.77 1.75 1.73
#  $ offset    : num  0.57 0.56 0.547
```

`A` is the known or estimated area of survey, `p` is availability given presence, `q` is detectability given availability.

NOTE: `offset` is `log(correction)`, `correction` = `A*p*q`, thus `offset=log(A) + log(p) + log(q)`.

### Step 4 Option 2. Calculate offsets for multiple species

Use a loop over multiple species:

```R
SPP <- getBAMspecieslist()
OFF <- matrix(0, nrow(x), length(SPP))
rownames(OFF) <- rownames(x) # your survey IDs here
colnames(OFF) <- SPP

for (spp in SPP) {
  cat(spp, "\n")
  flush.console()
  o <- make_off(spp, x, useMeth)
  OFF[,spp] <- o$offset
}
str(OFF)
##num [1, 1:151] 0.1365 0.9699 0.0643 0.5917 -0.3132 ...
## - attr(*, "dimnames")=List of 2
##  ..$ : chr "17"
##  ..$ : chr [1:151] "ALFL" "AMCR" "AMGO" "AMPI" ...
```

*This repository originates from the archived <https://github.com/ABbiodiversity/recurring> project.*
