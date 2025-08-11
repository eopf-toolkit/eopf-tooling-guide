# Issues

The following issues are required to support EOPF Zarr data in QGIS via STAC. This document contains short summaries only and will be linked to full issues once created.

## QGIS

### Stop Stripping Double Quotes

QGIS strips all double quotes from raster file paths within the Add Raster Layer dialog. This is a problem because the GDAL Zarr driver splits paths on colons `:` _unless_ they are wrapped by double quotes. If the QGIS user provides a Zarr path that includes double quoted colons (e.g. `ZARR:"/vsicurl/http://...domain:port/path:with:colons/..."`) it should work with the GDAL Zarr driver, but QGIS breaks the path in a way that cannot be avoided by escaping or additional quoting which results in GDAL erroring when it attempts to load the path `http`.

### Zarr as Cloud-Optimised Media Type

QGIS's core STAC integration does not currently consider Zarr a cloud-optimised format and therefore does not support streaming of Zarr STAC assets by dragging and dropping these assets into the layer pane. It instead offers downloads, which largely fail due to a number of other issues referenced here and which also does not make much sense for Zarr data. This issue should either 
1) disable downloads for Zarr data, or 
2) modify which files QGIS attempts to download - i.e. only fetching metadata JSON files and not data chunks.

### Add /vsicurl/ Prefix to HTTP Zarr Paths

When a STAC item asset references a Zarr-type dataset with a HTTP-hosted path QGIS should automatically add a `/vsicurl/` prefix to ensure GDAL correctly works with this dataset.

## GDAL

### More Intelligent Zarr Path Splitting

Similar problem to [QGIS / Stop Stripping Double Quotes](#stop-stripping-double-quotes). GDAL should not naively split paths on all non-double quoted colons. Other options could include more intelligently identifying individual paths (which may be colon-separated in a string of multiple paths) or supporting other quoting approaches. Further discussion is required here to understand the reasoning behind colon-splitting. This issue is less important if [QGIS / Stop Stripping Double Quotes](#stop-stripping-double-quotes) is resolved, but still worthy of investigation.

## EOPF Zarr

### Include _CRS Property in Zarr Metadata

GDAL relies on a `_CRS` property in Zarr metadata to identify the appropriate spatial reference but this property is missing from EOPF Zarr. An example of EOPF Zarr metadata with this property added is available and will be linked within this issue. This issue will be opened in CPM as previously requested.

## EODC Object Store

### Support HTTP Directory Listing

Support HTTP directory listing within the EODC object store referenced by EOPF STAC assets hrefs. GDAL's `vsicurl` driver partially relies on directory listings when attempting to identify a dataset's type and select the appropriate driver.

### Support Directory Path Slash Redirects

STAC asset hrefs currently reference directory paths without a trailing slash. Either [EODC Object Store / Support HTTP Directory Listing](#support-http-directory-listing) should return a listing from paths both with and without trailing slashes or directory paths without trailing slashes should redirect to directory paths with trailing slashes.

## EOPF STAC

### STAC Item Assets for More Arrays

Currently EOPF STAC items offer only a confusing subset of a Zarr store's arrays as top-level assets, while other arrays can only be discovered by interrogating the Zarr store. This forces the user to make potentially uninformed decisions about which arrays may be worth accessing within Zarr readers (e.g. QGIS). Decisions around which Zarr arrays require top-level STAC item assets should be more consistent and more intuitive from the user's perspective. This should also address performance problems observed with QGIS attempting to identify and list all available Zarr arrays from a Zarr store.

