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
Zarr array; we remove `product_url` to get a better idea of what each
array is.

``` r
zarr_store <- product_url %>%
  zarr_overview(as_data_frame = TRUE) %>%
  mutate(array = str_remove(path, product_url)) %>%
  relocate(array, .before = path)

zarr_store
```

                                                     array
    1                           /conditions/geometry/angle
    2                            /conditions/geometry/band
    3                        /conditions/geometry/detector
    4                 /conditions/geometry/mean_sun_angles
    5   /conditions/geometry/mean_viewing_incidence_angles
    6                      /conditions/geometry/sun_angles
    7        /conditions/geometry/viewing_incidence_angles
    8                               /conditions/geometry/x
    9                               /conditions/geometry/y
    10        /conditions/mask/detector_footprint/r10m/b02
    11        /conditions/mask/detector_footprint/r10m/b03
    12        /conditions/mask/detector_footprint/r10m/b04
    13        /conditions/mask/detector_footprint/r10m/b08
    14          /conditions/mask/detector_footprint/r10m/x
    15          /conditions/mask/detector_footprint/r10m/y
    16        /conditions/mask/detector_footprint/r20m/b05
    17        /conditions/mask/detector_footprint/r20m/b06
    18        /conditions/mask/detector_footprint/r20m/b07
    19        /conditions/mask/detector_footprint/r20m/b11
    20        /conditions/mask/detector_footprint/r20m/b12
    21        /conditions/mask/detector_footprint/r20m/b8a
    22          /conditions/mask/detector_footprint/r20m/x
    23          /conditions/mask/detector_footprint/r20m/y
    24        /conditions/mask/detector_footprint/r60m/b01
    25        /conditions/mask/detector_footprint/r60m/b09
    26        /conditions/mask/detector_footprint/r60m/b10
    27          /conditions/mask/detector_footprint/r60m/x
    28          /conditions/mask/detector_footprint/r60m/y
    29        /conditions/mask/l1c_classification/r60m/b00
    30          /conditions/mask/l1c_classification/r60m/x
    31          /conditions/mask/l1c_classification/r60m/y
    32        /conditions/mask/l2a_classification/r20m/scl
    33          /conditions/mask/l2a_classification/r20m/x
    34          /conditions/mask/l2a_classification/r20m/y
    35        /conditions/mask/l2a_classification/r60m/scl
    36          /conditions/mask/l2a_classification/r60m/x
    37          /conditions/mask/l2a_classification/r60m/y
    38                /conditions/meteorology/cams/aod1240
    39                 /conditions/meteorology/cams/aod469
    40                 /conditions/meteorology/cams/aod550
    41                 /conditions/meteorology/cams/aod670
    42                 /conditions/meteorology/cams/aod865
    43               /conditions/meteorology/cams/bcaod550
    44               /conditions/meteorology/cams/duaod550
    45          /conditions/meteorology/cams/isobaricInhPa
    46               /conditions/meteorology/cams/latitude
    47              /conditions/meteorology/cams/longitude
    48                 /conditions/meteorology/cams/number
    49               /conditions/meteorology/cams/omaod550
    50               /conditions/meteorology/cams/ssaod550
    51                   /conditions/meteorology/cams/step
    52               /conditions/meteorology/cams/suaod550
    53                /conditions/meteorology/cams/surface
    54                   /conditions/meteorology/cams/time
    55             /conditions/meteorology/cams/valid_time
    56                      /conditions/meteorology/cams/z
    57         /conditions/meteorology/ecmwf/isobaricInhPa
    58              /conditions/meteorology/ecmwf/latitude
    59             /conditions/meteorology/ecmwf/longitude
    60                   /conditions/meteorology/ecmwf/msl
    61                /conditions/meteorology/ecmwf/number
    62                     /conditions/meteorology/ecmwf/r
    63                  /conditions/meteorology/ecmwf/step
    64               /conditions/meteorology/ecmwf/surface
    65                  /conditions/meteorology/ecmwf/tco3
    66                  /conditions/meteorology/ecmwf/tcwv
    67                  /conditions/meteorology/ecmwf/time
    68                   /conditions/meteorology/ecmwf/u10
    69                   /conditions/meteorology/ecmwf/v10
    70            /conditions/meteorology/ecmwf/valid_time
    71                  /measurements/reflectance/r10m/b02
    72                  /measurements/reflectance/r10m/b03
    73                  /measurements/reflectance/r10m/b04
    74                  /measurements/reflectance/r10m/b08
    75                    /measurements/reflectance/r10m/x
    76                    /measurements/reflectance/r10m/y
    77                  /measurements/reflectance/r20m/b01
    78                  /measurements/reflectance/r20m/b02
    79                  /measurements/reflectance/r20m/b03
    80                  /measurements/reflectance/r20m/b04
    81                  /measurements/reflectance/r20m/b05
    82                  /measurements/reflectance/r20m/b06
    83                  /measurements/reflectance/r20m/b07
    84                  /measurements/reflectance/r20m/b11
    85                  /measurements/reflectance/r20m/b12
    86                  /measurements/reflectance/r20m/b8a
    87                    /measurements/reflectance/r20m/x
    88                    /measurements/reflectance/r20m/y
    89                  /measurements/reflectance/r60m/b01
    90                  /measurements/reflectance/r60m/b02
    91                  /measurements/reflectance/r60m/b03
    92                  /measurements/reflectance/r60m/b04
    93                  /measurements/reflectance/r60m/b05
    94                  /measurements/reflectance/r60m/b06
    95                  /measurements/reflectance/r60m/b07
    96                  /measurements/reflectance/r60m/b09
    97                  /measurements/reflectance/r60m/b11
    98                  /measurements/reflectance/r60m/b12
    99                  /measurements/reflectance/r60m/b8a
    100                   /measurements/reflectance/r60m/x
    101                   /measurements/reflectance/r60m/y
    102                       /quality/atmosphere/r10m/aot
    103                       /quality/atmosphere/r10m/wvp
    104                         /quality/atmosphere/r10m/x
    105                         /quality/atmosphere/r10m/y
    106                       /quality/atmosphere/r20m/aot
    107                       /quality/atmosphere/r20m/wvp
    108                         /quality/atmosphere/r20m/x
    109                         /quality/atmosphere/r20m/y
    110                       /quality/atmosphere/r60m/aot
    111                       /quality/atmosphere/r60m/wvp
    112                         /quality/atmosphere/r60m/x
    113                         /quality/atmosphere/r60m/y
    114                   /quality/l2a_quicklook/r10m/band
    115                    /quality/l2a_quicklook/r10m/tci
    116                      /quality/l2a_quicklook/r10m/x
    117                      /quality/l2a_quicklook/r10m/y
    118                   /quality/l2a_quicklook/r20m/band
    119                    /quality/l2a_quicklook/r20m/tci
    120                      /quality/l2a_quicklook/r20m/x
    121                      /quality/l2a_quicklook/r20m/y
    122                   /quality/l2a_quicklook/r60m/band
    123                    /quality/l2a_quicklook/r60m/tci
    124                      /quality/l2a_quicklook/r60m/x
    125                      /quality/l2a_quicklook/r60m/y
    126                             /quality/mask/r10m/b02
    127                             /quality/mask/r10m/b03
    128                             /quality/mask/r10m/b04
    129                             /quality/mask/r10m/b08
    130                               /quality/mask/r10m/x
    131                               /quality/mask/r10m/y
    132                             /quality/mask/r20m/b05
    133                             /quality/mask/r20m/b06
    134                             /quality/mask/r20m/b07
    135                             /quality/mask/r20m/b11
    136                             /quality/mask/r20m/b12
    137                             /quality/mask/r20m/b8a
    138                               /quality/mask/r20m/x
    139                               /quality/mask/r20m/y
    140                             /quality/mask/r60m/b01
    141                             /quality/mask/r60m/b09
    142                             /quality/mask/r60m/b10
    143                               /quality/mask/r60m/x
    144                               /quality/mask/r60m/y
    145                     /quality/probability/r20m/band
    146                      /quality/probability/r20m/cld
    147                      /quality/probability/r20m/snw
    148                        /quality/probability/r20m/x
    149                        /quality/probability/r20m/y
                                                                                                                                                                                                                               path
    1                           https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/geometry/angle
    2                            https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/geometry/band
    3                        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/geometry/detector
    4                 https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/geometry/mean_sun_angles
    5   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/geometry/mean_viewing_incidence_angles
    6                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/geometry/sun_angles
    7        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/geometry/viewing_incidence_angles
    8                               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/geometry/x
    9                               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/geometry/y
    10        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r10m/b02
    11        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r10m/b03
    12        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r10m/b04
    13        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r10m/b08
    14          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r10m/x
    15          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r10m/y
    16        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r20m/b05
    17        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r20m/b06
    18        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r20m/b07
    19        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r20m/b11
    20        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r20m/b12
    21        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r20m/b8a
    22          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r20m/x
    23          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r20m/y
    24        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r60m/b01
    25        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r60m/b09
    26        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r60m/b10
    27          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r60m/x
    28          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/detector_footprint/r60m/y
    29        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/l1c_classification/r60m/b00
    30          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/l1c_classification/r60m/x
    31          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/l1c_classification/r60m/y
    32        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/l2a_classification/r20m/scl
    33          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/l2a_classification/r20m/x
    34          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/l2a_classification/r20m/y
    35        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/l2a_classification/r60m/scl
    36          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/l2a_classification/r60m/x
    37          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/mask/l2a_classification/r60m/y
    38                https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/aod1240
    39                 https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/aod469
    40                 https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/aod550
    41                 https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/aod670
    42                 https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/aod865
    43               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/bcaod550
    44               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/duaod550
    45          https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/isobaricInhPa
    46               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/latitude
    47              https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/longitude
    48                 https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/number
    49               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/omaod550
    50               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/ssaod550
    51                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/step
    52               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/suaod550
    53                https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/surface
    54                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/time
    55             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/valid_time
    56                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/cams/z
    57         https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/isobaricInhPa
    58              https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/latitude
    59             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/longitude
    60                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/msl
    61                https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/number
    62                     https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/r
    63                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/step
    64               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/surface
    65                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/tco3
    66                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/tcwv
    67                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/time
    68                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/u10
    69                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/v10
    70            https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/conditions/meteorology/ecmwf/valid_time
    71                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/b02
    72                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/b03
    73                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/b04
    74                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/b08
    75                    https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/x
    76                    https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/y
    77                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b01
    78                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b02
    79                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b03
    80                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b04
    81                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b05
    82                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b06
    83                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b07
    84                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b11
    85                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b12
    86                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b8a
    87                    https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/x
    88                    https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/y
    89                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b01
    90                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b02
    91                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b03
    92                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b04
    93                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b05
    94                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b06
    95                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b07
    96                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b09
    97                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b11
    98                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b12
    99                  https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b8a
    100                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/x
    101                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/y
    102                       https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r10m/aot
    103                       https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r10m/wvp
    104                         https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r10m/x
    105                         https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r10m/y
    106                       https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r20m/aot
    107                       https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r20m/wvp
    108                         https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r20m/x
    109                         https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r20m/y
    110                       https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r60m/aot
    111                       https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r60m/wvp
    112                         https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r60m/x
    113                         https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/atmosphere/r60m/y
    114                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r10m/band
    115                    https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r10m/tci
    116                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r10m/x
    117                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r10m/y
    118                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r20m/band
    119                    https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r20m/tci
    120                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r20m/x
    121                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r20m/y
    122                   https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r60m/band
    123                    https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r60m/tci
    124                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r60m/x
    125                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/l2a_quicklook/r60m/y
    126                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r10m/b02
    127                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r10m/b03
    128                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r10m/b04
    129                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r10m/b08
    130                               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r10m/x
    131                               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r10m/y
    132                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r20m/b05
    133                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r20m/b06
    134                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r20m/b07
    135                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r20m/b11
    136                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r20m/b12
    137                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r20m/b8a
    138                               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r20m/x
    139                               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r20m/y
    140                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r60m/b01
    141                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r60m/b09
    142                             https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r60m/b10
    143                               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r60m/x
    144                               https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/mask/r60m/y
    145                     https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/probability/r20m/band
    146                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/probability/r20m/cld
    147                      https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/probability/r20m/snw
    148                        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/probability/r20m/x
    149                        https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/quality/probability/r20m/y
        nchunks  data_type compressor          dim    chunk_dim
    1         1 unicode224      blosc            2            2
    2         1  unicode96      blosc           13           13
    3         1      int64      blosc            5            5
    4         1    float64      blosc            2            2
    5         1    float64      blosc        13, 2        13, 2
    6         1    float64      blosc    2, 23, 23    2, 23, 23
    7         2    float64      blosc 13, 5, 2.... 7, 5, 2,....
    8         1      int64      blosc           23           23
    9         1      int64      blosc           23           23
    10       36      uint8      blosc 10980, 10980   1830, 1830
    11       36      uint8      blosc 10980, 10980   1830, 1830
    12       36      uint8      blosc 10980, 10980   1830, 1830
    13       36      uint8      blosc 10980, 10980   1830, 1830
    14        1      int64      blosc        10980        10980
    15        1      int64      blosc        10980        10980
    16       36      uint8      blosc   5490, 5490     915, 915
    17       36      uint8      blosc   5490, 5490     915, 915
    18       36      uint8      blosc   5490, 5490     915, 915
    19       36      uint8      blosc   5490, 5490     915, 915
    20       36      uint8      blosc   5490, 5490     915, 915
    21       36      uint8      blosc   5490, 5490     915, 915
    22        1      int64      blosc         5490         5490
    23        1      int64      blosc         5490         5490
    24       36      uint8      blosc   1830, 1830     305, 305
    25       36      uint8      blosc   1830, 1830     305, 305
    26       36      uint8      blosc   1830, 1830     305, 305
    27        1      int64      blosc         1830         1830
    28        1      int64      blosc         1830         1830
    29       36      uint8      blosc   1830, 1830     305, 305
    30        1      int64      blosc         1830         1830
    31        1      int64      blosc         1830         1830
    32       36      uint8      blosc   5490, 5490     915, 915
    33        1      int64      blosc         5490         5490
    34        1      int64      blosc         5490         5490
    35       36      uint8      blosc   1830, 1830     305, 305
    36        1      int64      blosc         1830         1830
    37        1      int64      blosc         1830         1830
    38        1    float32      blosc         9, 9         9, 9
    39        1    float32      blosc         9, 9         9, 9
    40        1    float32      blosc         9, 9         9, 9
    41        1    float32      blosc         9, 9         9, 9
    42        1    float32      blosc         9, 9         9, 9
    43        1    float32      blosc         9, 9         9, 9
    44        1    float32      blosc         9, 9         9, 9
    45        1    float64       <NA>                          
    46        1    float64      blosc            9            9
    47        1    float64      blosc            9            9
    48        1      int64       <NA>                          
    49        1    float32      blosc         9, 9         9, 9
    50        1    float32      blosc         9, 9         9, 9
    51        1      int64       <NA>                          
    52        1    float32      blosc         9, 9         9, 9
    53        1    float64       <NA>                          
    54        1      int64       <NA>                          
    55        1      int64       <NA>                          
    56        1    float32      blosc         9, 9         9, 9
    57        1    float64       <NA>                          
    58        1    float64      blosc            9            9
    59        1    float64      blosc            9            9
    60        1    float32      blosc         9, 9         9, 9
    61        1      int64       <NA>                          
    62        1    float32      blosc         9, 9         9, 9
    63        1      int64       <NA>                          
    64        1    float64       <NA>                          
    65        1    float32      blosc         9, 9         9, 9
    66        1    float32      blosc         9, 9         9, 9
    67        1      int64       <NA>                          
    68        1    float32      blosc         9, 9         9, 9
    69        1    float32      blosc         9, 9         9, 9
    70        1      int64       <NA>                          
    71       36     uint16      blosc 10980, 10980   1830, 1830
    72       36     uint16      blosc 10980, 10980   1830, 1830
    73       36     uint16      blosc 10980, 10980   1830, 1830
    74       36     uint16      blosc 10980, 10980   1830, 1830
    75        1      int64      blosc        10980        10980
    76        1      int64      blosc        10980        10980
    77       36     uint16      blosc   5490, 5490     915, 915
    78       36     uint16      blosc   5490, 5490     915, 915
    79       36     uint16      blosc   5490, 5490     915, 915
    80       36     uint16      blosc   5490, 5490     915, 915
    81       36     uint16      blosc   5490, 5490     915, 915
    82       36     uint16      blosc   5490, 5490     915, 915
    83       36     uint16      blosc   5490, 5490     915, 915
    84       36     uint16      blosc   5490, 5490     915, 915
    85       36     uint16      blosc   5490, 5490     915, 915
    86       36     uint16      blosc   5490, 5490     915, 915
    87        1      int64      blosc         5490         5490
    88        1      int64      blosc         5490         5490
    89       36     uint16      blosc   1830, 1830     305, 305
    90       36     uint16      blosc   1830, 1830     305, 305
    91       36     uint16      blosc   1830, 1830     305, 305
    92       36     uint16      blosc   1830, 1830     305, 305
    93       36     uint16      blosc   1830, 1830     305, 305
    94       36     uint16      blosc   1830, 1830     305, 305
    95       36     uint16      blosc   1830, 1830     305, 305
    96       36     uint16      blosc   1830, 1830     305, 305
    97       36     uint16      blosc   1830, 1830     305, 305
    98       36     uint16      blosc   1830, 1830     305, 305
    99       36     uint16      blosc   1830, 1830     305, 305
    100       1      int64      blosc         1830         1830
    101       1      int64      blosc         1830         1830
    102      36     uint16      blosc 10980, 10980   1830, 1830
    103      36     uint16      blosc 10980, 10980   1830, 1830
    104       1      int64      blosc        10980        10980
    105       1      int64      blosc        10980        10980
    106      36     uint16      blosc   5490, 5490     915, 915
    107      36     uint16      blosc   5490, 5490     915, 915
    108       1      int64      blosc         5490         5490
    109       1      int64      blosc         5490         5490
    110      36     uint16      blosc   1830, 1830     305, 305
    111      36     uint16      blosc   1830, 1830     305, 305
    112       1      int64      blosc         1830         1830
    113       1      int64      blosc         1830         1830
    114       1      int64      blosc            3            3
    115     108      uint8      blosc 3, 10980.... 1, 1830,....
    116       1      int64      blosc        10980        10980
    117       1      int64      blosc        10980        10980
    118       1      int64      blosc            3            3
    119     108      uint8      blosc 3, 5490,....  1, 915, 915
    120       1      int64      blosc         5490         5490
    121       1      int64      blosc         5490         5490
    122       1      int64      blosc            3            3
    123     108      uint8      blosc 3, 1830,....  1, 305, 305
    124       1      int64      blosc         1830         1830
    125       1      int64      blosc         1830         1830
    126      36      uint8      blosc 10980, 10980   1830, 1830
    127      36      uint8      blosc 10980, 10980   1830, 1830
    128      36      uint8      blosc 10980, 10980   1830, 1830
    129      36      uint8      blosc 10980, 10980   1830, 1830
    130       1      int64      blosc        10980        10980
    131       1      int64      blosc        10980        10980
    132      36      uint8      blosc   5490, 5490     915, 915
    133      36      uint8      blosc   5490, 5490     915, 915
    134      36      uint8      blosc   5490, 5490     915, 915
    135      36      uint8      blosc   5490, 5490     915, 915
    136      36      uint8      blosc   5490, 5490     915, 915
    137      36      uint8      blosc   5490, 5490     915, 915
    138       1      int64      blosc         5490         5490
    139       1      int64      blosc         5490         5490
    140      36      uint8      blosc   1830, 1830     305, 305
    141      36      uint8      blosc   1830, 1830     305, 305
    142      36      uint8      blosc   1830, 1830     305, 305
    143       1      int64      blosc         1830         1830
    144       1      int64      blosc         1830         1830
    145       1      int64       <NA>                          
    146      36      uint8      blosc   5490, 5490     915, 915
    147      36      uint8      blosc   5490, 5490     915, 915
    148       1      int64      blosc         5490         5490
    149       1      int64      blosc         5490         5490

This shows us the path to access the Zarr array, the number of chunks it
contains, the type of data, as well as its dimensions and chunking
structure. For example, for the `measurements/reflectance/r10m/b02`
array:

``` r
zarr_store %>%
  filter(array == "/measurements/reflectance/r10m/b02") %>%
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
b02_r10m <- zarr_store %>%
  filter(array == "/measurements/reflectance/r10m/b02") %>%
  pull(path)

b02_r10m %>%
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

# Zarr structure

The above overview tells us that the data is two-dimensional, with
dimensions 10980 x 10980. Zarr data is split up into *chunks*, which are
smaller, independent piece of the larger array. Chunks can be accessed
individually, without loading the entire array. In this case, there are
36 chunks in total, with 6 along each of the dimensions, each of size
1830 x 1830.

To read in Zarr data, we use `read_zarr_array()`, and can pass a list to
the `index` argument, describing which elements we want to extract,
along each dimension. Since this array is two-dimensional, we can think
of the dimensions as rows and columns of the data. For example, to
select the first 10 rows and the first 5 columns:

``` r
b02_r10m %>%
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
b02_r10m %>%
  read_zarr_array(list(8425:8430, 1:5))
```

         [,1] [,2] [,3] [,4] [,5]
    [1,] 9512 9456 9424 9360 9296
    [2,] 9488 9456 9352 9296 9216
    [3,] 9440 9392 9264 9216 9136
    [4,] 9400 9328 9328 9304 9160
    [5,] 9432 9376 9368 9272 9240
    [6,] 9440 9400 9336 9336 9352

TODO: Use the info Tom did
(https://github.com/eopf-toolkit/eopf-tooling-guide/blob/EOPF-48-tutorial-2/docs/tutorials/stac_zarr/python/eopf_stac_zarr_xarray.md#variables-and-attributes)
to describe what these units actually are etc.

With EOPF data, some measurements are available at multiple dimensions.
For example, we can see that the B02 spectral band is available at 10m,
20m, and 60m resolution:

``` r
b02 <- zarr_store %>%
  filter(str_starts(array, "/measurements/reflectance"), str_ends(array, "b02"))

b02
```

                                   array
    1 /measurements/reflectance/r10m/b02
    2 /measurements/reflectance/r20m/b02
    3 /measurements/reflectance/r60m/b02
                                                                                                                                                                                                             path
    1 https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r10m/b02
    2 https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r20m/b02
    3 https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m/b02
      nchunks data_type compressor          dim  chunk_dim
    1      36    uint16      blosc 10980, 10980 1830, 1830
    2      36    uint16      blosc   5490, 5490   915, 915
    3      36    uint16      blosc   1830, 1830   305, 305

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

# WIP —-

Want to look at owi:

``` r
item <- stac("https://stac.core.eopf.eodc.eu/") %>%
  collections(collection_id = "sentinel-1-l2-ocn") %>%
  items(feature_id = "S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971") %>%
  get_request()
```

``` r
owi_asset <- item$assets$owi

owi_asset[["title"]]
```

    [1] "Ocean Wind field"

``` r
zarr_store <- item %>%
  assets_select(asset_names = "product") %>%
  assets_url() %>%
  zarr_overview(as_data_frame = TRUE)
  
owi <- zarr_store %>%
  filter(str_starts(path, owi_asset$href)) %>%
  mutate(
    variable = str_remove(path, owi_asset$href),
    variable = str_remove(variable, "/")
  ) %>%
  relocate(variable, .before = path)

owi
```

            variable
    1       latitude
    2      longitude
    3 wind_direction
    4     wind_speed
                                                                                                                                                                                                                                                                path
    1       https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202506-s01siwocn/04/products/cpm_v256/S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971.zarr/owi/S01SIWOCN_20250604T193923_0025_A340_C971_0762FA_VV/measurements/latitude
    2      https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202506-s01siwocn/04/products/cpm_v256/S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971.zarr/owi/S01SIWOCN_20250604T193923_0025_A340_C971_0762FA_VV/measurements/longitude
    3 https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202506-s01siwocn/04/products/cpm_v256/S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971.zarr/owi/S01SIWOCN_20250604T193923_0025_A340_C971_0762FA_VV/measurements/wind_direction
    4     https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202506-s01siwocn/04/products/cpm_v256/S1A_IW_OCN__2SDV_20250604T193923_20250604T193948_059501_0762FA_C971.zarr/owi/S01SIWOCN_20250604T193923_0025_A340_C971_0762FA_VV/measurements/wind_speed
      nchunks data_type compressor      dim chunk_dim
    1       1   float32      blosc 167, 255  167, 255
    2       1   float32      blosc 167, 255  167, 255
    3       1   float32      blosc 167, 255  167, 255
    4       1   float32      blosc 167, 255  167, 255

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

Visualisation with stars, first convert to curvilinear grid, common in
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

![](eopf_zarr.markdown_strict_files/figure-markdown_strict/unnamed-chunk-7-1.png)
