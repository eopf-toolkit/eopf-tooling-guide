# STAC Item Zarr Assets

Among STAC item assets with the `application/vnd+zarr` type ("Zarr assets") one asset represents a top-level Zarr group in a Zarr store. This asset has both `data` and `metadata` roles and the title "EOPF Product" ("product asset"). The product asset supports access within xarray as a DataTree via its `xarray:open_datatree_kwargs` property. The product asset's href property ends in ".zarr". All Zarr data related to the STAC item can be accessed from the product asset.

In addition to the product asset, STAC items provide several assets representing groups and arrays nested within the top-level Zarr group.

> [!NOTE]
> Examples in the following text are primarily drawn from Sentinel 2 STAC items. Where assets are identified with a `/`-prefixed path, such as `/measurements/reflectance/r10m`, this path is the suffix after ".zarr" in the asset's href property (and also the name of the corresponding group or array within the product asset's xarray DataTree). For example:
> 
> `"href": "https://...zarr/measurements/reflectance/r10m"`


## Dataset Assets

Assets with `data` and `dataset` roles ("dataset assets") represent Zarr groups nested beneath the top-level "EOPF Product" group. These groups contain aligned Zarr arrays, which share common dimensions and coordinates. For example the Zarr group represented by the `/measurements/reflectance/r10m` asset contains Zarr arrays `b02`, `b03`, `b04`, and `b08`. Dataset assets include a `xarray:open_dataset_kwargs` property to support accessing them in xarray as a Dataset.


## Data Assets

Assets with the `data` role and not a corresponding `metadata` or `dataset` role ("data assets") represent individual Zarr arrays nested by Zarr groups. For example the Zarr array represented by the `/measurements/reflectance/r10m/b02` data asset is nested within the Zarr group represented by the `/measurements/reflectance/r10m` dataset asset. These Zarr arrays can be accessed within xarray via the product asset DataTree, the nesting dataset asset Dataset, or via the data asset's `href` property as a DataArray with `xarray.open_dataarray`.

> [!NOTE]
> At the time of writing https://github.com/EOPF-Sample-Service/eopf-stac/issues/26, https://github.com/EOPF-Sample-Service/eopf-stac/issues/25, and https://github.com/EOPF-Sample-Service/eopf-stac/issues/24 prevent dataset and data assets from being accessed directly in xarray. At this time Zarr groups and Zarr arrays must be accessed via the product asset's DataArray.


## Data Asset Availability

Where multiple Zarr groups provide Zarr arrays with the same measurement at different spatial resolutions, the STAC item only provides the Zarr array from the Zarr group with the finest spatial resolution as a data asset. For example the Zarr array `b8a` (narrow NIR band) exists in Zarr groups represented by the `/measurements/reflectance/r20m` and `/measurements/reflectance/r60m` dataset assets, however `/measurements/reflectance/r20m/b8a` is the only `b8a` data asset in the STAC item.


## Dataset Asset Availability

If all of a Zarr group's arrays are referenced by data assets, the STAC item does not include a dataset asset for the group. If only some of a Zarr group's arrays are represented by data assets then there will be a dataset asset for the group. For example the `/quality/atmosphere/r10m/aot` and `/quality/atmosphere/r10m/wvp` data assets represent all (2) arrays in the Zarr group `/quality/atmosphere/r10m` and this group is not represented by a dataset asset. However, `/measurements/reflectance/r60m/b09` is the only Zarr array within the 
`/measurements/reflectance/r60m` Zarr group with a data asset, because other Zarr arrays in this Zarr group are already represented by finer resolution data assets. As a result a `/measurements/reflectance/r60m` dataset asset is provided.
