# EOPF Tooling Guide

This repository serves as the central guide for all libraries and plugins developed to support the Earth Observation Processing Framework (EOPF) Zarr format for Sentinel data.

## Overview

The EOPF Toolkit provides a suite of libraries and plugins to facilitate working with Sentinel data in the new EOPF Zarr format. These tools enable seamless integration with existing workflows across multiple programming languages and applications.

## Available Tools

| Tools                   | Description                               | Status         | Repository                                                              |
| ----------------------- | ----------------------------------------- | -------------- | ----------------------------------------------------------------------- |
| STAC + Zarr             | EOPF Zarr Access from STAC                | In Development | [STAC+Zarr](docs/tutorials/stac_zarr)                                   |
| GDAL Zarr Driver        | Enhanced GDAL driver for EOPF Zarr        | Planned        | TBD                                                                     |
| QGIS Plugin             | Native QGIS integration for EOPF Zarr     | Planned        | TBD                                                                     |
| R Integration           | R libraries for EOPF Zarr access          | Planned        | TBD                                                                     |
| Julia Integration       | Julia packages for EOPF Zarr              | Planned        | TBD                                                                     |
| TiTiler Multidim        | Multidimensional data support for TiTiler | In Development | [titiler-multidim](https://github.com/developmentseed/titiler-multidim) |
| Stackstac Optimizations | Enhanced Stackstac for EOPF               | Planned        | TBD                                                                     |


## Getting Started

- [Overview of EOPF Tooling](docs/getting-started/overview.md)

## Documentation

- [Plugin Documentation](docs/plugins/)
- [Tutorials](docs/tutorials/)
- [FAQ](docs/faq.md)

## Development Roadmap

See our [roadmap](roadmap.md) for the planned development timeline and upcoming features.

## Contributing

We welcome contributions to improve the documentation and examples. Please see our [contributing guidelines](CONTRIBUTING.md).

## Related Resources

- [EOPF 101](https://github.com/sentinels-eopf-toolkit/eopf-101) - Educational content for learning about EOPF
- [EOPF Sample Service](https://eopf.copernicus.eu) - Access to EOPF data

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.