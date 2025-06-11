# Access and Analyze EOPF STAC Zarr Data with R


# Table of Contents

-   [Introduction](#introduction)
-   [Prerequisites](#prerequisites)
    -   [Dependencies](#dependencies)

# Introduction

This tutorial will explore how to access and analyze Zarr data from the
[EOPF Sample Service STAC
catalog](https://stac.browser.user.eopf.eodc.eu/) programmatically using
R.

# Prerequisites

An R environment is required to follow this tutorial, with R version \>=
4.1.0. We recommend using either
[RStudio](https://posit.co/download/rstudio-desktop/) or
[Positron](https://posit.co/products/ide/positron/) (or a cloud
computing environment) and making use of [RStudio
projects](https://support.posit.co/hc/en-us/articles/200526207-Using-RStudio-Projects)
for a self-contained coding environment.

## Dependencies

We will use the `rstac` package (for accessing the STAC catalog) and the
`tidyverse` package (for data manipulation) in this tutorial. You can
install them directly from CRAN:

``` r
install.packages("rstac")
install.packages("tidyverse")
```

We will also use the `Rarr` package. It must be installed from
Bioconductor, so first install the `BiocManager` package:

``` r
install.packages("BiocManager")
```

Then, use this package to install `Rarr`:

``` r
BiocManager::install("Rarr")
```

Finally, load the packages into your environment:

``` r
library(rstac)
library(tidyverse)
library(Rarr)
```

## Fixes to the `Rarr` package

We will use functions from the `Rarr` package to read and analyze Zarr
data. Unfortunately, there is currently a bug in this package, causing
it to parse the EOPF Sample Service data URLs incorrectly – there is a
[pull request](https://github.com/grimbough/Rarr/pull/21) open to fix
this. In the meantime, we will write our own version of this URL parsing
function and use it instead of the one in `Rarr`.

``` r
.url_parse_other <- function(url) {
  parsed_url <- httr::parse_url(url)
  bucket <- gsub(
    x = parsed_url$path, pattern = "^/?([[a-z0-9\\:\\.-]*)/.*",
    replacement = "\\1", ignore.case = TRUE
  )
  object <- gsub(
    x = parsed_url$path, pattern = "^/?([a-z0-9\\:\\.-]*)/(.*)",
    replacement = "\\2", ignore.case = TRUE
  )
  hostname <- paste0(parsed_url$scheme, "://", parsed_url$hostname)

  if (!is.null(parsed_url$port)) {
    hostname <- paste0(hostname, ":", parsed_url$port)
  }

  res <- list(
    bucket = bucket,
    object = object,
    region = "auto",
    hostname = hostname
  )
  return(res)
}

assignInNamespace(".url_parse_other", .url_parse_other, ns = "Rarr")
```

This function overwrites the existing one in `Rarr`, and allows us to
continue with the analysis.

If you try to run some of the examples below and receive a timeout
error, please ensure that you have run the above code block.

# Accessing Zarr data from the STAC Catalog

The first step of accessing Zarr data is to understand the assets within
the EOPF Sample Service STAC catalog. The [first tutorial](TODO) goes
into detail on this, so we recommend reviewing it if you have not
already.

For the first part of this tutorial, we will be using data from the
[Sentinel-2 Level-2A
Collection](https://stac.browser.user.eopf.eodc.eu/collections/sentinel-2-l2a).
We fetch the “product” asset under a given item, and can look at its
URL:

``` r
item <- stac("https://stac.core.eopf.eodc.eu/") %>%
  collections(collection_id = "sentinel-1-l2-ocn") %>%
  items(feature_id = "S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971") %>%
  get_request()

product <- item %>%
  assets_select(asset_names = "product")

product_url <- product %>%
  assets_url()

product_url
```

    [1] "https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202506-s01siwocn/04/products/cpm_v256/S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971.zarr"

The product is the “top level” Zarr asset, which shows us the full Zarr
store:

``` r
zarr_store <- zarr_overview(product_url, as_data_frame = TRUE) %>%
  as_tibble()
```

Want to look at owi:

``` r
owi_asset <- item$assets$owi

owi_asset[["title"]]
```

    [1] "Ocean Wind field"

``` r
owi <- zarr_store %>%
  filter(str_starts(path, owi_asset$href)) %>%
  mutate(
    variable = str_remove(path, owi_asset$href),
    variable = str_remove(variable, "/")
  ) %>%
  relocate(variable, .before = path)

owi
```

    # A tibble: 4 × 7
      variable       path               nchunks data_type compressor dim   chunk_dim
      <chr>          <chr>                <dbl> <chr>     <chr>      <lis> <list>   
    1 latitude       https://objects.e…       1 float32   blosc      <int> <int [2]>
    2 longitude      https://objects.e…       1 float32   blosc      <int> <int [2]>
    3 wind_direction https://objects.e…       1 float32   blosc      <int> <int [2]>
    4 wind_speed     https://objects.e…       1 float32   blosc      <int> <int [2]>

``` r
owi %>%
  filter(variable == "latitude") %>%
  pull(path) %>%
  zarr_overview()
```

    Type: Array
    Path: https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202506-s01siwocn/04/products/cpm_v256/S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971.zarr/owi/S01SIWOCN_20250604T193923_0025_A340_C971_0762FA_VV/measurements/latitude/
    Shape: 167 x 255
    Chunk Shape: 167 x 255
    No. of Chunks: 1 (1 x 1)
    Data Type: float32
    Endianness: little
    Compressor: blosc

``` r
owi %>%
  filter(variable == "longitude") %>%
  pull(path) %>%
  zarr_overview()
```

    Type: Array
    Path: https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202506-s01siwocn/04/products/cpm_v256/S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971.zarr/owi/S01SIWOCN_20250604T193923_0025_A340_C971_0762FA_VV/measurements/longitude/
    Shape: 167 x 255
    Chunk Shape: 167 x 255
    No. of Chunks: 1 (1 x 1)
    Data Type: float32
    Endianness: little
    Compressor: blosc

Reading data in. These are small and only have one chunk, we can read
them in quickly:

``` r
owi_lat <- owi %>%
  filter(variable == "latitude") %>%
  pull(path) %>%
  read_zarr_array()

owi_lat[1:5, 1:5]
```

             [,1]     [,2]     [,3]     [,4]     [,5]
    [1,] 30.26237 30.26406 30.26576 30.26746 30.26917
    [2,] 30.27138 30.27308 30.27478 30.27648 30.27818
    [3,] 30.28039 30.28209 30.28379 30.28549 30.28719
    [4,] 30.28940 30.29110 30.29280 30.29450 30.29620
    [5,] 30.29842 30.30012 30.30182 30.30351 30.30521

``` r
owi_long <- owi %>%
  filter(variable == "longitude") %>%
  pull(path) %>%
  read_zarr_array()

owi_lat[1:5, 1:5]
```

             [,1]     [,2]     [,3]     [,4]     [,5]
    [1,] 30.26237 30.26406 30.26576 30.26746 30.26917
    [2,] 30.27138 30.27308 30.27478 30.27648 30.27818
    [3,] 30.28039 30.28209 30.28379 30.28549 30.28719
    [4,] 30.28940 30.29110 30.29280 30.29450 30.29620
    [5,] 30.29842 30.30012 30.30182 30.30351 30.30521

``` r
owi_wind_direction <- owi %>%
  filter(variable == "wind_direction") %>%
  pull(path) %>%
  read_zarr_array()

owi_wind_direction[1:5, 1:5]
```

             [,1]     [,2]     [,3]     [,4]     [,5]
    [1,] 87.10201 85.10722 80.11242 87.11762 80.12283
    [2,] 87.10078 87.10600 88.11120 83.11641 86.12161
    [3,] 89.09956 81.10477 82.10999 88.11519 88.12040
    [4,] 87.09834 83.10355 84.10876 82.11398 81.11919
    [5,] 83.09712 88.10233 83.10755 86.11276 85.11797

Visualization with stars, first convert to curvilinear grid, common in
satellite data.

``` r
library(stars)
```

    Loading required package: sf

    Linking to GEOS 3.13.0, GDAL 3.8.5, PROJ 9.5.1; sf_use_s2() is TRUE

``` r
# Assume:
# - owi_dir: matrix of wind direction (167 × 255)
# - owi_long: matrix of longitude (167 × 255)
# - owi_lat: matrix of latitude (167 × 255)

# Step 1: Create stars object with wind data
s <- st_as_stars(wind = owi_wind_direction)

s <- st_as_stars(s, curvilinear = list(X1 = owi_long, X2 = owi_lat))

plot(s, as_points = FALSE, axes = TRUE, breaks = "equal", border = NA)
```

![](eopf_zarr.markdown_strict_files/figure-markdown_strict/unnamed-chunk-6-1.png)
