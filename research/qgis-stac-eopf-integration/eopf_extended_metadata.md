# EOPF Extended Metadata

This document demonstrates GDAL behaviour with EOPF metadata with and without changes recommended in the [Summary](./summary.md#recommended-approach) document. It intentionally excludes any testing of the [GDAL EOPF plugin](https://github.com/EOPF-Sample-Service/GDAL-ZARR-EOPF/) when reviewing GDAL behaviour.

## Data Access

This repo does not provide access to the dataset that is tested. The modified and unmodified Zarr stores are stored privately to avoid publishing additional data on ESA's behalf. Steps to download the unmodified dataset, and to modify it, are described below.

## Test Dataset

Tests focus on the [sentinel-2-l2a/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252](https://stac.core.eopf.eodc.eu/collections/sentinel-2-l2a/items/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252) STAC item and its `product` asset, whose `href` property links to the Zarr store at [this directory](https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr).

The following command was used to download the STAC item.

```sh
curl -o /path/to/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.json https://stac.core.eopf.eodc.eu/collections/sentinel-2-l2a/items/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252
```

The following script was used to download the Zarr store, using the arguments commented at the top.

```python
# python -m download_product "sentinel-2-l2a" "S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252" /path/to/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr

# download_product.py
import xarray as xr
from pystac_client import Client


def fetch(collection_id: str, item_id: str, output_path: str) -> None:
    client = Client.open("https://stac.core.eopf.eodc.eu/")
    collection = client.get_collection(collection_id=collection_id)
    item = collection.get_item(id=item_id)
    dt_asset = item.assets["product"]
    dt = xr.open_datatree(
        dt_asset.href,
        **dt_asset.extra_fields["xarray:open_datatree_kwargs"],
    )
    dt.to_zarr(output_path)


if __name__ == "__main__":
    from argparse import ArgumentParser

    parser = ArgumentParser()
    parser.add_argument(
        "collection",
        type=str,
    )
    parser.add_argument(
        "item",
        type=str,
    )
    parser.add_argument(
        "output_path",
        type=str,
    )
    args = parser.parse_args()
    fetch(
        collection_id=args.collection, item_id=args.item, output_path=args.output_path
    )

```

## Unmodified Dataset

The unmodified dataset is simply the files downloaded in [Test Dataset](#test-dataset). For simplicity they will be referred to as /unmodified.json and /unmodified.zarr.

## Modified Dataset

The following modifications were made to the test dataset. For simplicity they will be referred to as /modified.json and /modified.zarr.

### Zarr Metadata

Changes were made to the `.zmetadata` file at the root of the Zarr store and to each array's `.zattrs` file to include the `_CRS` property expected by GDAL.

Within `.zattrs` files the `_CRS` property was added as a root-level property, i.e.

```json
{
    "_CRS": { "url": "https://www.opengis.net/def/crs/EPSG/0/32626" },
    "_ARRAY_DIMENSIONS": [
        ...
    ],
    ...
```

Within `.zmetadata` the same property was added to each `"metadata"."[[array name]]".".zattrs"` entry, as demonstrated by [this example dataset](https://github.com/zarr-developers/geozarr-spec/issues/36#issuecomment-1934953474) in the GeoZarr Spec repo.

## Tests

### gdalinfo, Zarr Store

```sh
gdalinfo /path/to/test-store.zarr
```

#### Unmodified Zarr Store

No spatial reference reported.

> <pre>Driver: Zarr/Zarr
> Files: /unmodified
> Size is 512, 512
> Subdatasets:
> SUBDATASET_1_NAME=ZARR:"/unmodified":/conditions/geometry/mean_viewing_incidence_angles
> ...</pre>

#### Modified Zarr Store

No spatial reference reported.

> <pre>Driver: Zarr/Zarr
> Files: /modified
> Size is 512, 512
> Subdatasets:
> SUBDATASET_1_NAME=ZARR:"/modified":/conditions/geometry/mean_viewing_incidence_angles
> ...</pre>

### gdalinfo, Zarr Group

```sh
gdalinfo /path/to/test-store.zarr/measurements/reflectance/r10m
```

#### Unmodified Zarr Group

No spatial reference reported.

> <pre>Driver: Zarr/Zarr
> Files: /unmodified/measurements/reflectance/r10m
> Size is 512, 512
> Subdatasets:
> SUBDATASET_1_NAME=ZARR:"/unmodified/measurements/reflectance/r10m":/b02
> ...</pre>

#### Modified Zarr Group

No spatial reference reported.

> <pre>Driver: Zarr/Zarr
> Files: /modified/measurements/reflectance/r10m
> Size is 512, 512
> Subdatasets:
> SUBDATASET_1_NAME=ZARR:"/modified/measurements/reflectance/r10m":/b02
> ...</pre>

### gdalinfo, Zarr Array

```sh
gdalinfo /path/to/test-store.zarr/measurements/reflectance/r10m/b08
```

#### Unmodified Zarr Array

Metadata relevant to spatial reference printed but not recognised.

> <pre>Driver: Zarr/Zarr
> Files: /unmodified/measurements/reflectance/r10m/b08/.zarray
>        /unmodified/measurements/reflectance/r10m/b08
> Size is 10980, 10980
> Origin = (399960.000000000000000,4600020.000000000000000)
> Pixel Size = (10.000000000000000,-10.000000000000000)
> Metadata:
>   _eopf_attrs={ "add_offset": -0.1, "coordinates": [ "x", "y" ], "dimensions": [ "y", "x" ], "dtype": "<u2", "fill_value": 0, "long_name": "BOA reflectance from MSI acquisition at spectral band b08 833.0 nm", "scale_factor": 0.0001, "units": "digital_counts" }
>   dtype=<u2
>   fill_value=0
>   long_name=BOA reflectance from MSI acquisition at spectral band b08 833.0 nm
>   proj:bbox={399960,4490220,509760,4600020}
>   proj:epsg=32626
> ...</pre>

#### Modified Zarr Array

Spatial reference correctly reported.

> <pre>Driver: Zarr/Zarr
> Files: /modified/measurements/reflectance/r10m/b08/.zarray
>        /modified/measurements/reflectance/r10m/b08
> Size is 10980, 10980
> Coordinate System is:
> PROJCRS["WGS 84 / UTM zone 26N",
>     BASEGEOGCRS["WGS 84",
>         ENSEMBLE["World Geodetic System 1984 ensemble",
> ...</pre>

#### Additional

Test behaviour indicates that the `.zattrs` section for each array in the consolidated metadata was not read in the [gdalinfo, Zarr Array](#gdalinfo-zarr-array) tests. Metadata appeared to be sourced from `.zattrs` files in the array's directory.
