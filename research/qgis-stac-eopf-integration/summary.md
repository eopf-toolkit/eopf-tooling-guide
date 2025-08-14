# QGIS STAC Zarr Approach

This document describes research outcomes and presumed development requirements to support EOPF Zarr data in QGIS via STAC. The [Issues](./issues.md) document identifies work that must be completed.

Earlier versions of this document focused on the [QGIS STAC Browser Plugin](https://plugins.qgis.org/plugins/qgis_stac/) and the [GDAL EOPF plugin](https://github.com/EOPF-Sample-Service/GDAL-ZARR-EOPF). Neither of these plugins are currently considered part of a viable solution. Consult this document's commit history for more information on those approaches.

-----

## Zarr Support in QGIS

QGIS provides some level of support for Zarr data via GDAL. GDAL supports Zarr from v3.4+, and v3.8+ supports the current Zarr V3 specification. At the time of writing the QGIS Long Term Release (LTR) version is v3.40 and the official build includes GDAL v3.3, which does not support Zarr. QGIS can be installed via Conda with a more recent version of GDAL (e.g. 3.10.2) if Zarr support is required.

This document works with more recent versions of QGIS, including 3.42 (on macOS) and 3.44 (on Debian Linux).

### Performance

GDAL Zarr performance depends on how data are accessed. If GDAL is given a local or remotely-hosted path to a single Zarr array it can demonstrate adequate performance. If GDAL is given the path to a remotely-hosted Zarr group, and required to identify all of the arrays within that group (this is what happens when QGIS is asked to add a Zarr group), then performance is a function of the size and complexity of the group. For example, identifying all arrays in a resolution-based group within EOPF Zarr data ([example](https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr/measurements/reflectance/r60m)) might take several seconds. Identifying all arrays in an EOPF Zarr store ([example](https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr)) might take several minutes.

GDAL issues sequential HTTP HEAD and GET requests to traverse a Zarr group's metadata and fetches initial chunks of each array, presumably to validate against metadata. The number of HTTP requests issued depends on the number of arrays within the group.

### GUI

As noted in [this QGIS issue](https://github.com/qgis/QGIS/issues/54240#issuecomment-1854618963) and [this zarr-developers issue](https://github.com/zarr-developers/geozarr-spec/issues/36#issuecomment-1934953474) it is possible to load some Zarr data via the QGIS "Add Raster Layer" dialog.

#### Data on Local Disk

Zarr data on local disk can be added as layers either by dragging & dropping from the Browser panel or via the Add Raster Layer dialog. 

##### Sample Dataset

The following screenshot shows the sample Zarr dataset referenced [here](https://github.com/zarr-developers/geozarr-spec/issues/36#issuecomment-1934953474) loaded in QGIS from local disk.

![QGIS Zarr local screenshot](./images/qgis-zarr-disk-some.zarr.png "QGIS showing sample Zarr data from local disk")

In this example QGIS correctly identifies the dataset's spatial reference. This behaviour is discussed in [Spatial Reference Metadata](#spatial-reference-metadata).

##### EOPF Dataset

The following screenshot shows Zarr data downloaded from the "product" asset's `href` in [this](https://stac.core.eopf.eodc.eu/collections/sentinel-2-l2a/items/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252) STAC item loaded in QGIS from local disk.

![QGIS Zarr local screenshot](./images/qgis-zarr-disk-eopf.png "QGIS showing EOPF Zarr data from local disk")

In this example QGIS fails to identify the dataset's spatial reference. This behaviour is discussed in [Spatial Reference Metadata](#spatial-reference-metadata).

#### Hosted Data

Support for remotely-hosted Zarr data in QGIS is less mature, including in 3.44. GDAL supports two methods of accessing HTTP-hosted Zarr data: 
- `/vsicurl/` prefix, and
- `ZARR:` prefix.

##### /vsicurl/

`VSICURL` is invoked either by giving QGIS's `File` source-type a prefixed path such as `/vsicurl/https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr` or giving QGIS's `URL` source-type a non-prefixed path such as `https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr`.

GDAL's `VSICURL` driver attempts to identify any files available within an HTTP path and delegates accessing those files to compatible GDAL drivers. As referenced in GDAL Zarr driver [documentation](https://gdal.org/en/stable/drivers/raster/zarr.html#dataset-name) `VSICURL` depends on directory listing within the HTTP server when passed a HTTP directory path. If directory listing is supported `VSICURL` identifies the Zarr driver as compatible with Zarr data and successfully accesses data using that driver. If directory listing is not supported `VSICURL` fails with an error.

The EODC object store used to publish EOPF Zarr data via HTTP does not support directory listing, meaning `/vsicurl/` is not a viable approach.

##### ZARR:

The `ZARR:` prefix is used in combination with the `/vsicurl/` prefix, e.g. `ZARR:"/vsicurl/https://objectstore.eodc.eu:2222/e05ab01a9d56408d82ac32d69a5aae2a:202505-s02msil2a/22/products/cpm_v256/S2B_MSIL2A_20250522T125039_N0511_R095_T26TML_20250522T133252.zarr"`. It avoids the need for HTTP directory listing and therefore _should_ work with HTTP servers that do not support directory listing.

The Zarr driver splits `ZARR:`-prefixed paths on colons (`:`) that are not surrounded by double quotes and assumes the `[1]` split element represents the source location. As a result it requires HTTP paths to be double-quoted as in the example above. QGIS strips double quotes from paths before passing them to GDAL. The Zarr driver errors when the resulting split `[1]` element is simply `https`. It is not possible to avoid this behaviour using escape characters or alternate quoting strategies.

Due to QGIS quote-stripping it is not possible to access HTTP-hosted Zarr using the `ZARR:` prefix.

## Spatial Reference Metadata

As described in the GDAL Zarr driver [documentation](https://gdal.org/en/stable/drivers/raster/zarr.html#srs-encoding) GDAL expects a `_CRS` attribute which uses one of a number of approaches to define a Zarr dataset's spatial referencing.

The [Sample Dataset](#sample-dataset) example's Zarr store includes a consolidated metadata `.zmetadata` file with the following property, which is the reason QGIS can correctly identify the dataset's spatial reference.

```JSON
"_CRS": {
    "url": "http://www.opengis.net/def/crs/EPSG/0/6933"
}
```

EOPF Zarr consolidated metadata, including that in the [EOPF Dataset](#eopf-dataset) example, does not provide a `_CRS` property. Instead it provides a collection of properties in different locations.

### other_metadata

`metadata..zattrs.other_metadata` provides some spatial reference information across all members of the Zarr store.

```JSON
{
  "metadata": {
    ".zattrs": {
      "other_metadata": {
        ...
        "horizontal_CRS_code": "EPSG:32633",
        "horizontal_CRS_name": "WGS84 / UTM zone 33N",
        ...
      }
    }
  }
}
```

### stac_discovery

`metadata..zattrs.stac_discovery.properties` provides STAC item properties supported by the [proj](https://stac-extensions.github.io/projection/v2.0.0/schema.json) extension.

```JSON
{
  "metadata": {
    ".zattrs": {
      "stac_discovery": {
        "properties": {
          ...
          "proj:bbox": [...],
          "proj:epsg": 32633,
          ...
        }
      }
    }
  }
}
```

### Array .zattrs

`metadata.*..zattrs`, which is repeated for each array in the store, provides similar proj-style attributes.

```JSON
{
  "metadata": {
    ...
    "conditions/mask/detector_footprint/r10m/b03/.zattrs": {
      "proj:bbox": [...],
      "proj:epsg": 32633,
      "proj:shape": [...],
      "proj:transform": [...],
      "proj:wkt2": "PROJCS[...]"
    }
    ...
  }
}
```

### Discussion

There is active discussion as part of ongoing GeoZarr specification development on how spatial reference metadata should be handled in Zarr.

### Diagram

The following diagram documents the ways in which Zarr data can succeed or fail in QGIS.

![Flow chart showing success and failure paths for Zarr in QGIS](./diagrams/exports/2025.08.10%20QGIS%20Add%20EOPF%20Zarr%20Manually.png "Zarr paths in QGIS")

### pyQGIS

An [earlier version](https://github.com/eopf-toolkit/eopf-tooling-guide/blob/ccda97d22435e8d2872558b4e4c5e2c893b37490/research/qgis-stac-eopf-integration/summary.md) of this document included a number of Python scripts to demonstrate attempts to load EOPF Zarr in QGIS. As the requirement is for EOPF Zarr data in QGIS, and not pyQGIS, these scripts are no longer considered significant.

## STAC

An [earlier version](https://github.com/eopf-toolkit/eopf-tooling-guide/blob/ccda97d22435e8d2872558b4e4c5e2c893b37490/research/qgis-stac-eopf-integration/summary.md) of this document focused on the [QGIS STAC API Browser plugin](https://plugins.qgis.org/plugins/qgis_stac/) and changes that could be applied in that project to better support EOPF STAC in QGIS. At the time of writing it appears that this plugin does not have a maintenance strategy and is unlikely to progress beyond its current state or address outstanding issues.

Since version 3.40 QGIS has included basic native STAC support. As of QGIS 3.44 native STAC capabilities are largely comparable to the plugin's capabilities. It appears that any further work to better support STAC in QGIS should be applied to the native STAC integration and not the plugin.

Similar to the plugin, native STAC support limits which asset types can be added as layers. The following quote from Lutra Consulting's [blog post](https://www.lutraconsulting.co.uk/blogs/stac-in-qgis) (supported by [this function in the QGIS source code](https://github.com/qgis/QGIS/blob/4d06cb8c8ea32df13cadd8576da9101e4aaa1517/src/core/stac/qgsstacasset.cpp#L61) suggests only recognised cloud-optimised assets can be added as (streaming) layers and all other asset types must be downloaded before use.

> If the item is a Cloud Optimized format (e.g. COG), you can simply drag and drop it in QGIS canvas to view it. If it is a flat file, you need to first download the item.

Based on this wording it is unclear how multiple cloud-optimised assets within a single item, or assets of varying types within an item, will be handled. STAC items with COG assets can be dragged and dropped into the Layers pane, while items without cloud-optimised assets cannot.

The following screenshot shows a STAC item's `visual` asset added as a layer.

![QGIS native STAC item COG](./images/qgis-stac-native-cog.png "QGIS native STAC integration and COG layer")

The EOPF Sample STAC service's `sentinel-2-l2a` collection's items do not include any COG assets. QGIS native STAC integration offers to download all of the item's assets, but all downloads except the `product-metadata` asset fail with 404 responses. `product-metadata` succeeds because it is the only asset whose `href` references a file rather than a directory. Similar to the STAC plugin, native STAC does not elegantly handle directory-based assets like Zarr stores.

In the case of EOPF Zarr data the attempt to download the various directory paths in asset `href`s fails with 404 because the EODC object store does not support directory listing. Testing shows that when directory listing is supported QGIS simply downloads the HTML listing response, which is no more useful to the user than a 404.

![QGIS native STAC EOPF item assets](./images/qgis-stac-native-eopf-assets.png "QGIS native STAC integration and EOPF item assets")

-----

## Recommended Approach

The [Issues](./issues.md) document identifies all work required as part of a recommended approach. Issues have been created across multiple affected repositories to address the various limitations outlined above and ultimately support EOPF Zarr in QGIS via STAC.
