# EOPF Tooling Guide

This repository serves as the central guide for all libraries and plugins developed to support the Earth Observation Processing Framework (EOPF) Zarr format for Sentinel data.

## Overview

The EOPF Toolkit provides a suite of libraries and plugins to facilitate working with Sentinel data in the new EOPF Zarr format. These tools enable seamless integration with existing workflows across multiple programming languages and applications.

## Available Tools

| Tools                   | Description                               | Status         | Repository                                                              |
| ----------------------- | ----------------------------------------- | -------------- | ----------------------------------------------------------------------- |
| Python Integration      | Python access to EOPF Zarr using STAC       | In Development | [Python](docs/tutorials/stac_zarr/python/eopf_stac_access.md)                                                 |
| GDAL Zarr Driver        | Enhanced GDAL driver for EOPF Zarr          | Planned        | TBD                                                                     |
| QGIS Plugin             | QGIS integration for EOPF Zarr              | Planned        | TBD                                                                     |
| R Integration           | R access to EOPF Zarr using STAC            | In Development | [R](/docs/tutorials/stac_zarr/R/eopf_stac_access.md)                                                    |
| Julia Integration       | Julia access to EOPF Zarr using STAC        | Planned        | TBD                                                                     |
| TiTiler Multidim        | Multidimensional data support for TiTiler | In Development | [titiler-multidim](https://github.com/developmentseed/titiler-multidim) |
| Stackstac Optimizations | Enhanced Stackstac for EOPF               | Planned        | TBD                                                                     |

## Getting Started

- [Overview of EOPF Tooling](docs/getting-started/overview.md)

## Documentation

- [Plugin Documentation](docs/plugins/)
- [Tutorials](docs/tutorials/)
  - [STAC and Zarr Concepts](docs/tutorials/stac_zarr.md)
- [FAQ](docs/faq.md)

## Contributing

We welcome contributions to improve the documentation and examples. Please see our [contributing guidelines](CONTRIBUTING.md).

## Related Resources

- [EOPF](https://eopf.copernicus.eu) - Overview of EOPF
- [EOPF 101](https://github.com/eopf-toolkit/eopf-101) - Educational content for learning about EOPF


## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.