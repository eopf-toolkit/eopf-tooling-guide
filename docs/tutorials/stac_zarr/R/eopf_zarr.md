# Access and Analyze EOPF STAC Zarr Data with R


# Table of Contents

-   [Introduction](#introduction)
-   [Prerequisites](#prerequisites)
    -   [Dependencies](#dependencies)
    -   [Fixes to the Rarr package](#fixes-to-the-rarr-package)
-   [Access Zarr data from the STAC
    catalog](#access-zarr-data-from-the-stac-catalog)
-   [Read Zarr data](#read-zarr-data)
    -   [Coordinates](#coordinates)
    -   [Different resolutions](#different-resolutions)

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

We will use the `rstac` package (for accessing the STAC catalog), the
`tidyverse` package (for data manipulation), and the `stars` package
(for working with spatiotemporal data) in this tutorial. You can install
them directly from CRAN:

``` r
install.packages("rstac")
install.packages("tidyverse")
install.packages("stars")
```

We will also use the `Rarr` package to read Zarr data. It must be
installed from Bioconductor, so first install the `BiocManager` package:

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
library(stars)
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

# Access Zarr data from the STAC Catalog

The first step of accessing Zarr data is to understand the assets within
the EOPF Sample Service STAC catalog. The [first
tutorial](./eopf_stac_access.qmd) goes into detail on this, so we
recommend reviewing it if you have not already.

For the first part of this tutorial, we will be using data from the
[Sentinel-2 Level-2A
Collection](https://stac.browser.user.eopf.eodc.eu/collections/sentinel-2-l2a).
We fetch the “product” asset under a given item, and can look at its
URL:

``` r
item <- stac("https://stac.core.eopf.eodc.eu/") %>%
  collections(collection_id = "sentinel-2-l2a") %>%
  items(feature_id = "S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252") %>%
  get_request()

product <- item %>%
  assets_select(asset_names = "product")

product_url <- product %>%
  assets_url()

product_url
```

    [1] "https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr"

The product is the “top level” Zarr asset, which contains the full Zarr
product hierarchy. We can use `zarr_overview()` to get an overview of
it, setting `as_data_frame` to `TRUE` so that we can see the entries in
a data frame instead of printed directly to the console. Each entry is a
Zarr array; we remove `product_url` from the path to get a better idea
of what each array is. Since this is something we will want to do
multiple times throughout the tutorial, we create a helper function for
this.

``` r
derive_store_array <- function(store, product_url) {
  store %>%
  mutate(array = str_remove(path, product_url)) %>%
  relocate(array, .before = path)
}

zarr_store <- product_url %>%
  zarr_overview(as_data_frame = TRUE) %>%
  derive_store_array(product_url)

zarr_store
```

    # A tibble: 149 × 7
       array                      path  nchunks data_type compressor dim   chunk_dim
       <chr>                      <chr>   <dbl> <chr>     <chr>      <lis> <list>   
     1 /conditions/geometry/angle http…       1 unicode2… blosc      <int> <int [1]>
     2 /conditions/geometry/band  http…       1 unicode96 blosc      <int> <int [1]>
     3 /conditions/geometry/dete… http…       1 int64     blosc      <int> <int [1]>
     4 /conditions/geometry/mean… http…       1 float64   blosc      <int> <int [1]>
     5 /conditions/geometry/mean… http…       1 float64   blosc      <int> <int [2]>
     6 /conditions/geometry/sun_… http…       1 float64   blosc      <int> <int [3]>
     7 /conditions/geometry/view… http…       2 float64   blosc      <int> <int [5]>
     8 /conditions/geometry/x     http…       1 int64     blosc      <int> <int [1]>
     9 /conditions/geometry/y     http…       1 int64     blosc      <int> <int [1]>
    10 /conditions/mask/detector… http…      36 uint8     blosc      <int> <int [2]>
    # ℹ 139 more rows

This shows us the path to access the Zarr array, the number of chunks it
contains, the type of data, as well as its dimensions and chunking
structure.

We can also look at overviews of individual arrays. First, let’s narrow
down to measurements taken at 10m resolution:

``` r
r10m <- zarr_store %>%
  filter(str_starts(array, "/measurements/reflectance/r10m/"))

r10m
```

    # A tibble: 6 × 7
      array                       path  nchunks data_type compressor dim   chunk_dim
      <chr>                       <chr>   <dbl> <chr>     <chr>      <lis> <list>   
    1 /measurements/reflectance/… http…      36 uint16    blosc      <int> <int [2]>
    2 /measurements/reflectance/… http…      36 uint16    blosc      <int> <int [2]>
    3 /measurements/reflectance/… http…      36 uint16    blosc      <int> <int [2]>
    4 /measurements/reflectance/… http…      36 uint16    blosc      <int> <int [2]>
    5 /measurements/reflectance/… http…       1 int64     blosc      <int> <int [1]>
    6 /measurements/reflectance/… http…       1 int64     blosc      <int> <int [1]>

Then, we select the B02 array and examine its dimensions and chuning:

``` r
r10m %>%
  filter(str_ends(array, "b02")) %>%
  select(path, nchunks, dim, chunk_dim) %>%
  as.list()
```

    $path
    [1] "https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/b02"

    $nchunks
    [1] 36

    $dim
    $dim[[1]]
    [1] 10980 10980


    $chunk_dim
    $chunk_dim[[1]]
    [1] 1830 1830

We can also see an overview of individual arrays using
`zarr_overview()`. With the default setting (where `as_data_frame` is
`FALSE`), this prints information on the array directly to the console,
in a more digestible way:

``` r
r10m_b02 <- r10m %>%
  filter(str_ends(array, "b02")) %>%
  pull(path)

r10m_b02 %>%
  zarr_overview()
```

    Type: Array
    Path: https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/b02/
    Shape: 10980 x 10980
    Chunk Shape: 1830 x 1830
    No. of Chunks: 36 (6 x 6)
    Data Type: uint16
    Endianness: little
    Compressor: blosc

The above overview tells us that the data is two-dimensional, with
dimensions 10980 x 10980. Zarr data is split up into *chunks*, which are
smaller, independent piece of the larger array. Chunks can be accessed
individually, without loading the entire array. In this case, there are
36 chunks in total, with 6 along each of the dimensions, each of size
1830 x 1830.

# Read Zarr data

To read in Zarr data, we use `read_zarr_array()`, and can pass a list to
the `index` argument, describing which elements we want to extract,
along each dimension. Since this array is two-dimensional, we can think
of the dimensions as rows and columns of the data. For example, to
select the first 10 rows and the first 5 columns:

``` r
r10m_b02 %>%
  read_zarr_array(list(1:10, 1:5))
```

          [,1] [,2] [,3] [,4] [,5]
     [1,]    0    0    0    0    0
     [2,]    0    0    0    0    0
     [3,]    0    0    0    0    0
     [4,]    0    0    0    0    0
     [5,]    0    0    0    0    0
     [6,]    0    0    0    0    0
     [7,]    0    0    0    0    0
     [8,]    0    0    0    0    0
     [9,]    0    0    0    0    0
    [10,]    0    0    0    0    0

Or, to select rows rows 8425 to 8430 and columns 1 to 5:

``` r
r10m_b02 %>%
  read_zarr_array(list(8425:8430, 1:5))
```

         [,1] [,2] [,3] [,4] [,5]
    [1,] 9512 9456 9424 9360 9296
    [2,] 9488 9456 9352 9296 9216
    [3,] 9440 9392 9264 9216 9136
    [4,] 9400 9328 9328 9304 9160
    [5,] 9432 9376 9368 9272 9240
    [6,] 9440 9400 9336 9336 9352

## Coordinates

Similarly, we can read in the `x` and `y` coordinates corresponding to
data at 10m resolution. These `x` and `y` coordinates do not correspond
to latitude and longitude–to understand the coordinate reference system
used in each data set, we access the “`proj:espg`” property of the STAC
item. In this case, the coordinate reference system is
[EPSG:32626](https://epsg.io/32626), which represents metres from the
UTM zone’s origin.

``` r
item[["properties"]][["proj:code"]]
```

    [1] "EPSG:32626"

We can see that `x` and `y` are one dimensional:

``` r
r10m_x <- r10m %>%
  filter(str_ends(array, "x")) %>%
  pull(path)

r10m_x %>%
  zarr_overview()
```

    Type: Array
    Path: https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/x/
    Shape: 10980
    Chunk Shape: 10980
    No. of Chunks: 1 (1)
    Data Type: int64
    Endianness: little
    Compressor: blosc

``` r
r10m_y <- r10m %>%
  filter(str_ends(array, "y")) %>%
  pull(path)

r10m_y %>%
  zarr_overview()
```

    Type: Array
    Path: https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/y/
    Shape: 10980
    Chunk Shape: 10980
    No. of Chunks: 1 (1)
    Data Type: int64
    Endianness: little
    Compressor: blosc

Which means that, when combined, they form a grid that describes the
location of each point in the 2-dimensional measurements, such as B02.
We will go into this more in the examples below.

The `x` and `y` dimensions can be read in using the same logic: by
describing which elements we want to extract. Since there is only one
dimension, we only need to supply one entry in the indexing list:

``` r
r10m_x %>%
  read_zarr_array(list(1:5))
```

    [1] 399965 399975 399985 399995 400005

Or, we can read in the whole array and view its first few values with
`head()`. Of course, reading in the whole array, rather than a small
section of it, will take longer.

``` r
r10m_x %>%
  read_zarr_array() %>%
  head(5)
```

    [1] 399965 399975 399985 399995 400005

``` r
r10m_y %>%
  read_zarr_array() %>%
  head(5)
```

    [1] 4600015 4600005 4599995 4599985 4599975

## Different resolutions

With EOPF data, some measurements are available at multiple resolutions.
For example, we can see that the B02 spectral band is available at 10m,
20m, and 60m resolution:

``` r
b02 <- zarr_store %>%
  filter(str_starts(array, "/measurements/reflectance"), str_ends(array, "b02"))

b02
```

    # A tibble: 3 × 7
      array                       path  nchunks data_type compressor dim   chunk_dim
      <chr>                       <chr>   <dbl> <chr>     <chr>      <lis> <list>   
    1 /measurements/reflectance/… http…      36 uint16    blosc      <int> <int [2]>
    2 /measurements/reflectance/… http…      36 uint16    blosc      <int> <int [2]>
    3 /measurements/reflectance/… http…      36 uint16    blosc      <int> <int [2]>

The resolution affects the dimensions of the data; when measurements are
taken at a higher resolution, there will be more data. We can see here
that there is less data for the 20m resolution than the 10m resolution
(recall, its dimensions are 10980 x 10980), and even less for the 60m
resolution:

``` r
b02 %>%
  filter(array == "/measurements/reflectance/r20m/b02") %>%
  pull(dim)
```

    [[1]]
    [1] 5490 5490

``` r
b02 %>%
  filter(array == "/measurements/reflectance/r60m/b02") %>%
  pull(dim)
```

    [[1]]
    [1] 1830 1830

# Examples

The following sections show examples from each of the Sentinel missions.

## Sentinel 2

Since we have already been looking at Sentinel 2 data above, we will
first continue with this example.

## Sentinel 1

The second example looks at [Sentinel 1 Level 2 Ocean (OCN)
data](https://stac.browser.user.eopf.eodc.eu/collections/sentinel-1-l2-ocn),
which consists of data for oceanographic study, such as monitoring sea
surface conditions, detecting oil spills, and studying ocean currents.
This example will show how to access and plot Wind Direction data.

First, select the relevant collection and item from STAC:

``` r
l2_ocn <- stac("https://stac.core.eopf.eodc.eu/") %>%
  collections(collection_id = "sentinel-1-l2-ocn") %>%
  items(feature_id = "S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971") %>%
  get_request()

l2_ocn
```

    ###Item
    - id: S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971
    - collection: sentinel-1-l2-ocn
    - bbox: xmin: -25.77445, ymin: 30.25712, xmax: -22.82115, ymax: 32.16941
    - datetime: 2025-06-04T19:39:23.099186Z
    - assets: osw, owi, rvl, product, product_metadata
    - item's fields: 
    assets, bbox, collection, geometry, id, links, properties, stac_extensions, stac_version, type

We can look at each of the assets’ to understand what the item contains:

``` r
l2_ocn %>%
  pluck("assets") %>%
  map("title")
```

    $osw
    [1] "Ocean Swell spectra"

    $owi
    [1] "Ocean Wind field"

    $rvl
    [1] "Surface Radial Velocity"

    $product
    [1] "EOPF Product"

    $product_metadata
    [1] "Consolidated Metadata"

We are interested in the “Ocean Wind field” data, and will hold onto the
“owi” key for now.

To access all of the `owi` data, we get the `"product"` asset and then
the full Zarr store, again using our helper function to extract array
information from the full array path:

``` r
l2_ocn_url <- l2_ocn %>%
  assets_select(asset_names = "product") %>%
  assets_url() 

l2_ocn_store <- l2_ocn_url %>%
  zarr_overview(as_data_frame = TRUE) %>%
  derive_store_array(l2_ocn_url)

l2_ocn_store
```

    # A tibble: 114 × 7
       array                      path  nchunks data_type compressor dim   chunk_dim
       <chr>                      <chr>   <dbl> <chr>     <chr>      <lis> <list>   
     1 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [3]>
     2 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [2]>
     3 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [2]>
     4 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [2]>
     5 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [2]>
     6 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [2]>
     7 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [5]>
     8 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [5]>
     9 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [3]>
    10 /osw/S01SIWOCN_20250604T1… http…       1 float32   blosc      <int> <int [3]>
    # ℹ 104 more rows

Next, we filter to access `owi` measurement data only:

``` r
l2_ocn_store %>%
  filter(str_starts(array, "/owi"), str_detect(array, "measurements"))
```

    # A tibble: 4 × 7
      array                       path  nchunks data_type compressor dim   chunk_dim
      <chr>                       <chr>   <dbl> <chr>     <chr>      <lis> <list>   
    1 /owi/S01SIWOCN_20250604T19… http…       1 float32   blosc      <int> <int [2]>
    2 /owi/S01SIWOCN_20250604T19… http…       1 float32   blosc      <int> <int [2]>
    3 /owi/S01SIWOCN_20250604T19… http…       1 float32   blosc      <int> <int [2]>
    4 /owi/S01SIWOCN_20250604T19… http…       1 float32   blosc      <int> <int [2]>

Since all of these arrays start with
`/owi/S01SIWOCN_20250604T193923_0025_A340_C971_0762FA_VV/measurements/`,
we can remove that to get a clearer idea of what each array is:

``` r
owi <- l2_ocn_store %>%
  filter(str_starts(array, "/owi"), str_detect(array, "measurements")) %>%
  mutate(array = str_remove(array, "/owi/S01SIWOCN_20250604T193923_0025_A340_C971_0762FA_VV/measurements/"))

owi
```

    # A tibble: 4 × 7
      array          path               nchunks data_type compressor dim   chunk_dim
      <chr>          <chr>                <dbl> <chr>     <chr>      <lis> <list>   
    1 latitude       https://objects.e…       1 float32   blosc      <int> <int [2]>
    2 longitude      https://objects.e…       1 float32   blosc      <int> <int [2]>
    3 wind_direction https://objects.e…       1 float32   blosc      <int> <int [2]>
    4 wind_speed     https://objects.e…       1 float32   blosc      <int> <int [2]>

We are interested in `wind_direction`, as well as the coordinate arrays
(`latitude` and `longitude`). We can get an overview of the arrays’
dimensions and structures:

``` r
owi %>%
  filter(array == "wind_direction") %>%
  pull(path) %>%
  zarr_overview()
```

    Type: Array
    Path: https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202506-s01siwocn/04/products/cpm_v256/S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971.zarr/owi/S01SIWOCN_20250604T193923_0025_A340_C971_0762FA_VV/measurements/wind_direction/
    Shape: 167 x 255
    Chunk Shape: 167 x 255
    No. of Chunks: 1 (1 x 1)
    Data Type: float32
    Endianness: little
    Compressor: blosc

``` r
owi %>%
  filter(array == "latitude") %>%
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
  filter(array == "longitude") %>%
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

Here, we can see that all of the arrays are of the same shape: 167 x
255, with only one chunk. Since these are small, we can read all of the
data in at once. This is done by *not* supplying any additional
arguments to `read_zarr_array()`:

``` r
owi_wind_direction <- owi %>%
  filter(array == "wind_direction") %>%
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

``` r
owi_lat <- owi %>%
  filter(array == "latitude") %>%
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
  filter(array == "longitude") %>%
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

Note that, unlike in the previous example with `x` and `y` coordinates
that we explored, both `longitude` and `latitude` are 2-dimensional
arrays, and they are not evenly spaced. Rather, the data grid is
**curvilinear** — it has grid lines that are not straight, and there is
a longitude and latitude for every pixel of the other layers (i.e.,
`wind_direction`). This format is very common in satellite data.

We use functions from the [`stars`
package](https://r-spatial.github.io/stars/), loaded earlier, to format
the data for visualisation. `stars` is specifically designed for
reading, manipulating, and plotting spatiotemporal data, such as
satellite data.

The function `st_as_stars()` is used to get our data into the correct
format for visualisation:

``` r
owi_stars <- st_as_stars(wind_direction = owi_wind_direction) %>%
  st_as_stars(curvilinear = list(X1 = owi_long, X2 = owi_lat))
```

Getting the data into this format is also beneficial because it allows
for a quick summary of the data and its attributes, providing
information such as the median and mean `wind_direction`, the number of
`NA`s, and information on the grid:

``` r
owi_stars
```

    stars object with 2 dimensions and 1 attribute
    attribute(s):
                        Min. 1st Qu.   Median     Mean  3rd Qu.     Max. NA's
    wind_direction  33.29902 57.9872 66.76632 65.45217 73.04303 91.13456  430
    dimension(s):
       from  to         refsys point                      values x/y
    X1    1 167 WGS 84 (CRS84) FALSE [167x255] -25.77,...,-22.83 [x]
    X2    1 255 WGS 84 (CRS84) FALSE   [167x255] 30.26,...,32.16 [y]
    curvilinear grid

Finally, we can plot this object:

``` r
plot(owi_stars, as_points = FALSE, axes = TRUE, breaks = "equal", border = NA)
```

![](eopf_zarr.markdown_strict_files/figure-markdown_strict/owi-plot-1.png)

## Sentinel 2

## Sentinel 3
