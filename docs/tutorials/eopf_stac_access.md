# EOPF STAC Metadata Access

## Table of Contents

## Prerequisites

A Python environment is required to follow this tutorial. This tutorial's dependencies require Python >= 3.10. A virtual environment is recommended.

### Dependencies

The following dependencies are required to follow this tutorial.

```sh
pip install "pystac-client>=0.8.6,<1.0"
```

## Discover Zarr Assets

### Imports

The following imports are required to complete all following steps. Not all imports are required by all steps.

```python
from typing import List, Optional, cast

import requests
from pystac import Collection, MediaType
from pystac_client import Client, CollectionClient
```

### Connect to the EOPF Sample Service STAC API

Create a connection to the EOPF Sample Service STAC API and verify the connection is successful.

```python
max_description_length = 100

eopf_stac_api_root_endpoint = "https://stac.core.eopf.eodc.eu/"
client = Client.open(url=eopf_stac_api_root_endpoint)
print(
    "Connected to Catalog {id}: {description}".format(
        id=client.id,
        description=client.description
        if len(client.description) <= max_description_length
        else client.description[: max_description_length - 3] + "...",
    )
)
```

### Browse Available Collections

Iterate through available collections and report some basic information about each.

> [!NOTE]
> An [outstanding issue (#18)](https://github.com/EOPF-Sample-Service/eopf-stac/issues/18) within the STAC API requires a workaround to successfully iterate through available collections. Affected code is commented appropriately.

```python
all_collections: Optional[List[Collection]] = None
# May fail due to #18
try:
    all_collections = [_ for _ in client.get_all_collections()]
    print(
        "* [https://github.com/EOPF-Sample-Service/eopf-stac/issues/18 appears to be resolved]"
    )
except Exception:
    print(
        "* [https://github.com/EOPF-Sample-Service/eopf-stac/issues/18 appears to not be resolved]"
    )

if all_collections is None:
    # Workaround for #18
    valid_collections: List[Collection] = []
    for collection_href in [link.absolute_href for link in client.get_child_links()]:
        collection_dict = requests.get(url=collection_href).json()
        try:
            valid_collections.append(Collection.from_dict(collection_dict))
        except Exception as e:
            if isinstance(e, TypeError) and "not subscriptable" in str(e).lower():
                # This exception is expected for some collections due to #18
                continue
            else:
                raise e
    all_collections = valid_collections

# Iterate over all available (valid) collections and report basic information.
for collection in all_collections:
    collection_parent = collection.get_parent()
    print("Collection {id}".format(id=collection.id))
    if collection_parent is not None:
        print(
            " - Child of {parent_id}".format(
                parent_id=collection_parent.id,
            )
        )
    # Do not print the entire description as it may be very long
    print(
        " - Description: {description}".format(
            description=collection.description
            if len(collection.description) <= max_description_length
            else collection.description[: max_description_length - 3] + "..."
        )
    )
```

### Search for Items

#### Spatial Extent (Bounding Box)

Search for items whose spatial extents intersect central Paris.

```python
bbox_search_results_sample = client.search(
    bbox=(
        2.23949,
        48.81410,
        2.43896,
        48.90546,
    ),
    max_items=1,
)
for item in bbox_search_results_sample.items():
    print(
        "bbox search result item ID: {id}, BBOX: [{bbox}]".format(
            id=item.id, bbox=item.bbox
        )
    )
```

#### Temporal Extent (Time Range)

Search for items whose datetime intersects a given day (UTC time).

```python
time_search_results_sample = client.search(
    datetime="2025-05-22T00:00:00Z/2025-05-22T23:59:59.999999Z", max_items=1
)
for item in time_search_results_sample.items():
    print(
        "time search result item ID: {id}, datetime: {datetime}".format(
            id=item.id, datetime=item.datetime
        )
    )
```

#### Platform

Search for items whose platform is 'sentinel-2b'.

```python
platform_search_results_sample = client.search(
    filter={"op": "eq", "args": [{"property": "platform"}, "sentinel-2b"]},
    filter_lang="cql2-json",
    max_items=1,
)
for item in platform_search_results_sample.items():
    print(
        "platform search result item ID: {id}, platform: {platform}".format(
            id=item.id, platform=item.properties["platform"]
        )
    )
```

#### Instruments

Search for items whose instruments include 'msi'.

```python
instruments_search_results_sample = client.search(
    filter={"op": "A_CONTAINS", "args": [{"property": "instruments"}, ["msi"]]},
    filter_lang="cql2-json",
    max_items=1,
)
for item in instruments_search_results_sample.items():
    print(
        "instruments search result item ID: {id}, instruments: {instruments}".format(
            id=item.id, instruments=item.properties["instruments"]
        )
    )
```

#### Combined Search Criteria

Search with all prior criteria combined.

```python
combined_search_results_sample = client.search(
    bbox=(
        2.23949,
        48.81410,
        2.43896,
        48.90546,
    ),
    datetime="2025-05-22T00:00:00Z/2025-05-22T23:59:59.999999Z",
    filter={
        "op": "and",
        "args": [
            {"op": "eq", "args": [{"property": "platform"}, "sentinel-2b"]},
            {"op": "A_CONTAINS", "args": [{"property": "instruments"}, ["msi"]]},
        ],
    },
    max_items=1,
)
for item in combined_search_results_sample.items():
    print(
        "combined search result item ID: {id}, BBOX: {bbox}, datetime: {datetime}, platform: {platform}, instruments: {instruments}".format(
            id=item.id,
            bbox=item.bbox,
            datetime=item.datetime,
            platform=item.properties["platform"],
            instruments=item.properties["instruments"],
        )
    )
```

### STAC Item Metadata

Extract STAC item metadata and identify Zarr assets.

```python
sentinel_2_l2a_collection = cast(
    CollectionClient, client.get_collection(collection_id="sentinel-2-l2a")
)
sample_item = sentinel_2_l2a_collection.get_item(
    id="S2B_MSIL2A_20250522T105619_N0511_R094_T32WPE_20250522T121018"
)
assert sample_item is not None, "Expected item does not exist"
print("Sample item {id}".format(id=sample_item.id))
print(" - Datetime: {datetime}".format(datetime=sample_item.datetime))
print(
    " - Processing Level: {processing_level}".format(
        processing_level=sample_item.properties["processing:level"]
    )
)
for asset_name, asset in sample_item.get_assets(media_type=MediaType.ZARR).items():
    print(
        " - Zarr asset {asset_name} at {asset_href}".format(
            asset_name=asset_name, asset_href=asset.href
        )
    )
```
