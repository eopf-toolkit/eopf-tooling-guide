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

TODO -\> is purrr needed?

We will use the `rstac` package (for accessing the STAC catalog) and the
`purrr` package (for data manipulation) in this tutorial. You can
install them directly from CRAN:

``` r
install.packages("rstac")
install.packages("purrr")
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
library(purrr)
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

The first step of accessing Zarr data is to find the correct asset
within the EOPF Sample Service STAC catalog. The [first tutorial](TODO)
goes into detail on this, so we recommend reviewing it if you have not
already.

For the first part of this tutorial, we will be using data from the
[Sentinel-2 Level-2A
Collection](https://stac.browser.user.eopf.eodc.eu/collections/sentinel-2-l2a).
We fetch the “product” asset under a given item, and can look at its
URL:

``` r
product <- stac("https://stac.core.eopf.eodc.eu/") %>%
  collections(collection_id = "sentinel-2-l2a") %>%
  items(feature_id = "S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252") %>%
  get_request() %>%
  assets_select(asset_names = "product")

product_url <- product %>%
  assets_url()

product_url
```

    [1] "https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr"

This URL is what we use to begin loading and analyzing the data.

# Exploring Zarr data

Zarr is a format that allows the storage of large, multidimensional
array data. The data is divided into subsets known as **chunks**, and
the Zarr format allows for efficient access to those chunks.

We cannot read all of the data in at once, nor is it typically
desirable. As we see on the [Sentinel-2 Level-2A Collection
page](https://stac.browser.user.eopf.eodc.eu/collections/sentinel-2-l2a),
the data is split up into multiple **resolutions** as well as multiple
**bands**, which contain the actual variable measurements, as well as
quality assurance bands, which help to identify and improve the accuracy
of the measurements.

The `zarr_overview()` function gives us an quick overview of the data:

``` r
zarr_overview(product_url)

#> Type: Group of Arrays
#> Path: https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/
#> Arrays:
#> ---
#>   Path: conditions/geometry/angle
#>   Shape: 2
#>   Chunk Shape: 2
#>   No. of Chunks: 1 (1)
#>   Data Type: unicode224
#>   Endianness: little
#>   Compressor: blosc
#> ---
#>   Path: conditions/geometry/band
#>   Shape: 13
#>   Chunk Shape: 13
#>   No. of Chunks: 1 (1)
#>   Data Type: unicode96
#>   Endianness: little
#>   Compressor: blosc
#> ---
#>   Path: conditions/geometry/detector
#>   Shape: 5
#>   Chunk Shape: 5
#>   No. of Chunks: 1 (1)
#>   Data Type: int64
#>   Endianness: little
#>   Compressor: blosc
```

^^ just a small amount of this, because it’s huge!

Some notes (for myself):

-   These are actually *groups* of arrays, not individual arrays
-   I think there is one array per band/resolution
-   Is this the “hierarchical” groups that Rarr says it cannot handle?

``` r
product_overview <- zarr_overview(product_url, as_data_frame = TRUE)

library(dplyr)
library(tidyr)
library(stringr)

product_overview <- product_overview %>% as_tibble()

# Remove main URL from path, split by / into hierarchies

zarr_hierarchies <- product_overview %>%
  mutate(
    path = str_remove(path, product_url),
    path = str_remove(path, "/")
  ) %>%
  separate(path, into = c("one", "two", "three", "four", "five"), sep = "/", fill = "right")

zarr_hierarchies %>%
  count(one)
```

    # A tibble: 3 × 2
      one              n
      <chr>        <int>
    1 conditions      70
    2 measurements    31
    3 quality         48

``` r
zarr_hierarchies %>%
  count(one, two)
```

    # A tibble: 8 × 3
      one          two               n
      <chr>        <chr>         <int>
    1 conditions   geometry          9
    2 conditions   mask             28
    3 conditions   meteorology      33
    4 measurements reflectance      31
    5 quality      atmosphere       12
    6 quality      l2a_quicklook    12
    7 quality      mask             19
    8 quality      probability       5

According to https://highway.esa.int/support/data-services/eopf-format,
an EOProduct consists of:

-   measurements
-   quality
-   conditions

so we see that is the case here

``` r
zarr_hierarchies %>%
  filter(one == "measurements", two == "reflectance") %>%
  count(three)
```

    # A tibble: 3 × 2
      three     n
      <chr> <int>
    1 r10m      6
    2 r20m     12
    3 r60m     13

This then contains the resolutions

``` r
zarr_hierarchies %>%
  filter(one == "measurements", two == "reflectance") %>%
  count(four)
```

    # A tibble: 14 × 2
       four      n
       <chr> <int>
     1 b01       2
     2 b02       3
     3 b03       3
     4 b04       3
     5 b05       2
     6 b06       2
     7 b07       2
     8 b08       1
     9 b09       1
    10 b11       2
    11 b12       2
    12 b8a       2
    13 x         3
    14 y         3

which then have the bands and x/y within.

But these things need to go together, e.g. to have the x and y for b01
measurements, and to also have the masks to e.g. remove cloudy pixels

``` r
zarr_hierarchies %>%
  filter(one == "quality", two == "mask") %>%
  distinct(three, four)
```

    # A tibble: 19 × 2
       three four 
       <chr> <chr>
     1 r10m  b02  
     2 r10m  b03  
     3 r10m  b04  
     4 r10m  b08  
     5 r10m  x    
     6 r10m  y    
     7 r20m  b05  
     8 r20m  b06  
     9 r20m  b07  
    10 r20m  b11  
    11 r20m  b12  
    12 r20m  b8a  
    13 r20m  x    
    14 r20m  y    
    15 r60m  b01  
    16 r60m  b09  
    17 r60m  b10  
    18 r60m  x    
    19 r60m  y    

So how do we read in the actual Zarr data? The
[docs](https://bioconductor.org/packages/release/bioc/vignettes/Rarr/inst/doc/Rarr.html#limitations-with-rarr)
say:

> If you know about Zarr arrays already, you’ll probably be aware they
> can be stored in hierarchical groups, where additional meta data can
> explain the relationship between the arrays. Currently, Rarr is not
> designed to be aware of these hierarchical Zarr array collections.
> However, the component arrays can be read individually by providing
> the path to them directly.

So will work on providing the direct paths instead.
