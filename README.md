MEOS — ARCHIVED SNAPSHOT
========================

> **⚠️ This repository is a snapshot of MEOS taken in April 2026 and is not maintained.**
>
> **MEOS is developed in [MobilityDB/MobilityDB](https://github.com/MobilityDB/MobilityDB/tree/master/meos), under `meos/`.** Its documentation is at [libmeos.org](https://libmeos.org).
>
> The code below predates six months of MEOS development; do not build from it.

**Mobility Engine, Open Source**

[![Test MEOS compilation](https://github.com/MobilityDB/MEOS/actions/workflows/meos.yml/badge.svg)](https://github.com/MobilityDB/MEOS/actions/workflows/meos.yml)

MEOS is a C library for manipulating temporal and spatio-temporal data. It provides the type system and operations that power [MobilityDB](https://github.com/MobilityDB/MobilityDB), and can also be used standalone — embedded in any C application or wrapped through one of its language bindings.

MEOS implements the [ISO 19141:2008](https://www.iso.org/standard/41445.html) standard *Geographic information — Schema for moving features*, with extensions for non-spatial attribute changes and for the *temporal gaps* that naturally arise when collecting mobility data (intervals during which the position of a moving object is not observed).

The name and design philosophy are inspired by [GEOS](https://libgeos.org/) (Geometry Engine, Open Source). The official project website is [libmeos.org](https://libmeos.org/).

What MEOS Provides
------------------

*   **Temporal types** for tracking how a value changes over time:
    *   `tbool`, `tint`, `tfloat`, `ttext` — temporal versions of base types
    *   `tgeompoint`, `tgeogpoint` — temporal geometry/geography points
    *   `tgeometry`, `tgeography` — temporal geometries/geographies (any shape)
    *   Optional: `tcbuffer` (circular buffers), `tnpoint` (network points), `tpose` (rigid-body poses), `trgeometry` (rigid geometries)

*   **Temporal subtypes** capturing different observation patterns:
    *   *Instant* — a single value at a single time
    *   *Sequence* — a contiguous interval of values, with linear or step interpolation
    *   *SequenceSet* — multiple disjoint sequences, naturally representing temporal gaps

*   **Bounding box types** — `tbox` (temporal numeric box) and `stbox` (spatio-temporal box) for fast indexing and pruning

*   **Set & span types** — `intset`, `bigintset`, `floatset`, `textset`, `dateset`, `tstzset`, plus `intspan`, `floatspan`, `datespan`, `tstzspan` and their span sets

*   **Operations** — construction, parsing, output (WKT, WKB, MF-JSON), aggregation, distance, spatial relationships, temporal tiling, simplification, and more

Project Structure
-----------------

```
MEOS/
├── include/        # Public headers (meos.h, meos_geo.h, meos_internal.h, ...)
├── src/            # Source files
│   ├── temporal/   #   Core temporal types and operations
│   ├── geo/        #   Spatio-temporal types and operations
│   ├── cbuffer/    #   Circular buffers (optional)
│   ├── npoint/     #   Network points (optional)
│   ├── pose/       #   Geoposes (optional)
│   └── rgeo/       #   Rigid geometries (optional)
├── postgres/       # Embedded PostgreSQL compatibility layer (date/time, utils)
├── postgis/        # Embedded PostGIS code (liblwgeom, ryu)
├── examples/       # Example programs (hello world, AIS ingestion, BerlinMOD, ...)
├── test/           # Standalone tests
└── tools/          # pkg-config template
```

MEOS is self-contained: it embeds the parts of PostgreSQL and PostGIS it needs, so the only external dependencies at runtime are GEOS, PROJ, JSON-C, and GSL.

Requirements
------------

*   Linux (other UNIX-like systems may work, but remain untested)
*   CMake >= 3.12
*   A C compiler (GCC or Clang)
*   GEOS >= 3.8
*   PROJ >= 6.1
*   JSON-C
*   GNU Scientific Library (GSL)

On Debian/Ubuntu:

```bash
sudo apt install build-essential cmake libgeos-dev libproj-dev libjson-c-dev libgsl-dev
```

Building & Installation
-----------------------

```bash
git clone https://github.com/MobilityDB/MEOS
mkdir MEOS/build
cd MEOS/build
cmake ..
make
sudo make install
```

The build produces `libmeos.so` (or `libmeos.dylib` on macOS) and installs the public headers (`meos.h`, `meos_geo.h`, `meos_internal.h`, ...) under `${CMAKE_INSTALL_INCLUDEDIR}`.

### Build options

| Option           | Default | Description                                                        |
|------------------|---------|--------------------------------------------------------------------|
| `CBUFFER`        | `OFF`   | Include circular buffer types                                      |
| `NPOINT`         | `ON`    | Include network point types                                        |
| `POSE`           | `OFF`   | Include geopose types                                              |
| `RGEO`           | `OFF`   | Include rigid geometry types (requires `POSE=ON`)                  |
| `BUILD_SHARED_LIBS` | `ON`  | Build as shared library; set `OFF` for a static library            |
| `DEBUG_EXPAND`   | `OFF`   | Print debug messages for expandable data structures                |

Example:

```bash
cmake -DCBUFFER=ON -DPOSE=ON ..
```

Hello, MEOS
-----------

```c
#include <stdio.h>
#include <stdlib.h>
#include <meos.h>
#include <meos_geo.h>

int main(void)
{
  meos_initialize();

  Temporal *t = tgeompoint_in("[POINT(1 1)@2000-01-01, POINT(2 2)@2000-01-02]");
  char *out = temporal_as_mfjson(t, true, 6, 0, NULL);
  printf("%s\n", out);

  free(out);
  free(t);
  meos_finalize();
  return 0;
}
```

Compile and run:

```bash
gcc -Wall -g -I/usr/local/include -o hello hello.c -L/usr/local/lib -lmeos
./hello
```

More example programs are under [`examples/`](examples/), including AIS ingestion pipelines, BerlinMOD-based trajectory processing, expandable data structures, simplification, and tiling.

Language Bindings
-----------------

Because MEOS exposes a stable C API, it is straightforward to call from other languages. Maintained bindings include:

*   **Python** — [PyMEOS](https://github.com/MobilityDB/PyMEOS) (CFFI)
*   **Java** — [JMEOS](https://github.com/MobilityDB/JMEOS) (JNR-FFI)
*   **Go** — [GoMEOS](https://github.com/MobilityDB/GoMEOS)
*   **Rust** — [meos.rs](https://github.com/MobilityDB/meos-rs)
*   **C#** — [MEOS.NET](https://github.com/MobilityDB/MEOS.NET)

Relationship to MobilityDB
--------------------------

MEOS is the core type system used by [MobilityDB](https://github.com/MobilityDB/MobilityDB), the PostgreSQL extension that brings temporal and spatio-temporal types to SQL. MobilityDB consumes MEOS as a git submodule and recompiles its sources with PostgreSQL-specific bindings.

If you want **SQL access** to temporal types, install MobilityDB. If you want to **embed** temporal types in a C application, a pipeline component, or a non-PostgreSQL system, use MEOS directly.

Documentation
-------------

*   Project website: [libmeos.org](https://libmeos.org/)
*   API reference and developer docs: [mobilitydb.github.io/MobilityDB](https://mobilitydb.github.io/MobilityDB/)
*   Textbook: Sakr, Vaisman, Zimányi — *[Mobility Data Science: From Data to Insights](https://link.springer.com/book/10.1007/978-3-031-82636-8)*, Springer 2025

License
-------

MEOS is released under the [PostgreSQL License](LICENSE.txt). It includes portions of PostGIS source code released under the GNU General Public License (GPLv2 or later).

Acknowledgements
----------------

MEOS is developed by the Computer & Decision Engineering Department of the [Université libre de Bruxelles](https://www.ulb.be/) (ULB), under the direction of [Prof. Esteban Zimányi](http://cs.ulb.ac.be/members/esteban/).

The MobilityDB/MEOS project has received funding from the European Union's [Horizon Europe](https://open-research-europe.ec.europa.eu/gateways/horizon-europe) research and innovation programme under grant agreements No 101070279 [MobiSpaces](https://mobispaces.eu/) and No 101093051 [EMERALDS](https://emeralds-horizon.eu/).
