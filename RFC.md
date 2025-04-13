# RFC: TrackingDataContainer — A Metadata-Aware Container for Tracking Data

## Summary

This RFC proposes the creation of a `TrackingDataContainer`: a shared, standardized, metadata-aware container for sports tracking data. It is backed by Apache Arrow and acts as the central interface between tools like Kloppy, Floodlight, BallRadar, and others.

The container combines high-performance, zero-copy data access with a rich metadata model, enabling efficient workflows, semantic column selection, and reproducibility. All packages in the ecosystem are expected to use this container as the primary I/O interface.

Note: it should probably extend the `TrackingDataset` from kloppy.

---

## Motivation

As tracking pipelines grow across football and other sports, the need for modularity, performance, and inter-package interoperability becomes critical.

Current challenges:

- Object-based tracking (e.g. Kloppy) doesn't scale well to large datasets
- Packages use inconsistent schemas and lack a shared representation
- Derived metrics and transforms aren’t reproducible or shareable
- Skeleton data, training data, and multi-team use cases need flexibility

A container backed by Apache Arrow, with enforced metadata and a shared schema, solves these problems while keeping compatibility with modern analytics engines (Polars, DuckDB, etc.).

---

## Goals

- Define a canonical, metadata-rich container for tracking data
- Standardize column naming for interoperability
- Ensure all packages use the container interface
- Enable fast, lazy access to Arrow or Polars objects
- Support derived metrics, frame materialization, and multiple data layers (e.g. skeleton)

### Non-goals

- Defining a full storage or lakehouse solution
- Handling raw input formats (delegated to Kloppy or custom parsers)

---

## Design Overview

The `TrackingDataContainer` wraps an Apache Arrow table and Kloppy-style metadata. It supports metadata-aware column access, derived metric registration, and engine-agnostic transformations (Polars, NumExpr, etc.).

Packages access the container via a consistent interface, with optional access to Arrow, Polars LazyFrames, or NumPy arrays.

```python
container = kloppy.load_as_container("data.json", "metadata.json")

# or:
from kloppy import secondspectrum
container = secondspectrum.load(...normal arguments..., as_container=True)

# or container must be fully backwards compatible (best option)

# Add ball speed
container.add_metric("ball_speed", expression="sqrt((ball.x - ball.x.shift(1))**2 + ...)", engine="polars")

# Frame materialization
frame = container.materialize_frame(timestamp=534.2)

# Use LazyFrame
df = container.lazy().select([...]).collect()
```

---

## Detailed Design

### Core Components

```python
class TrackingDataContainer:
    def arrow(self) -> pa.Table
    def lazy(self) -> pl.LazyFrame
    def selector: ColumnSelector  # e.g. selector.player("home", 7).x
    def add_metric(name: str, expression: str | pl.Expr, engine: str)
    def add_metrics(list_of_exprs: list[pl.Expr])
    def get_column(...)
    def materialize_frame(timestamp: float)
```

### Column Naming Convention

We propose to align column naming with the [Common Data Format (CDF)](https://doi.org/10.48550/arXiv.2401.01882) standard as published by Anzer et al. (2025). This avoids reinventing the wheel and increases interoperability with current and future tooling that supports this standard.

Example conventions based on CDF:

| Concept   | Column Name                                |
|-----------|---------------------------------------------|
| Ball      | `ball_x`, `ball_y`, `ball_z`                |
| Player    | `teams/players/<player_id>_x`, `_y`         |
| Skeleton  | `teams/players/<player_id>/<limb>_x`        |
| Metrics   | `package/metric_name`                       |
| Time      | `timestamp`, `frame_id`, `period`           |

These conventions follow:

- Snake_case field names
- British spelling (e.g. "colour")
- Metric or object first, followed by descriptor
- Units described in metadata, not column name

By adhering to CDF, the `TrackingDataContainer` will be interoperable with other ecosystems (e.g. research, federation-level analytics) and allow users to mix data from different vendors more reliably.

### Metadata

- Required for all sessions
- Includes teams, players, field size, orientation, coordinate system
- Tracks added metrics, including provenance
- Defines optional logical "layers" (e.g. tracking, skeleton, predictions)

### Storage & Computation Model

The container is designed to support **lazy evaluation** and **filter pushdown** using Apache Arrow-compatible formats like Parquet and Iceberg.

- **Lazy Evaluation**: All transformations (e.g. metric calculations, filters) are applied lazily using Polars' `LazyFrame` or Arrow compute APIs.
- **Filter Pushdown**: When reading from Parquet or Iceberg, the container supports pushing column and row filters down to the storage engine, dramatically improving performance for large datasets.
- **Column Projection**: Only the necessary columns are loaded or materialized, based on selectors or expressions.
- **Iceberg Integration (Future Direction)**: Iceberg can serve as the underlying storage format, offering versioning, schema evolution, and partition pruning. The container will provide adapters to read from and write to Iceberg tables.

This design ensures the container can be scaled from local processing to lakehouse-style infrastructure with minimal friction.

---

## Use Cases / Examples

- Load tracking data and add derived metrics
- Run smoothing or pitch control via external packages
- Convert skeleton data to tracking data (e.g. using center of mass)
- Visualize or export a single frame for debugging
- Export Polars dataframe with semantic selectors

---

## Rationale & Alternatives

### Why not just Arrow?
- Arrow is fast and standard, but lacks semantics
- Metadata enables safe, interpretable, and sport-agnostic usage

### Why enforce the container?
- To guarantee consistency across packages
- To avoid schema drift and partial implementations

### Why CDF and flat schema?
- CDF provides a well-vetted, community-driven naming standard
- Flat schema increases compatibility and simplifies filter pushdown and projection
- Structs may be supported later as an internal representation

---

## Compatibility

- Packages must use `TrackingDataContainer` for I/O
- Read-only tools may optionally operate on Arrow, if they follow the schema
- Kloppy will provide `load_as_container(...)` as the default loader

---

## Future Directions

- Iceberg support with partitioning, time-travel, and pushdown
- Event and prediction layers as Arrow sub-tables or column groups
- Multi-sensor support (e.g. GPS + camera)
- JSON-LD-style metadata for validation and discovery

---

## Open Questions

- Should metrics be namespaced (`package/metric`)?
- Should struct-based layouts be supported now or later?
- Do we support event data natively or as a separate container?
- Does `TrackingDataContainer` need to extend from kloppy `Dataset`? It probably should
- Add a FrameBuilder? (using PyArrow `ArrayBuilder` - not supported in Python yet - see https://github.com/apache/arrow/issues/20529 )
- Related tickets:
  - [Refactor serializer options into own component](https://github.com/PySport/kloppy/issues/10) -> Describes a FrameBuilder approach
  - [Refactor tracking data model](https://github.com/PySport/kloppy/pull/377)

---

## Conclusion

The `TrackingDataContainer` provides a shared, fast, and extensible foundation for sports tracking pipelines. It aligns with modern data practices (Arrow), adopts the CDF standard for naming and schema, encourages inter-package reuse, and ensures semantic correctness through enforced metadata.

We propose making this the default container for all tools in the ecosystem.

**Feedback welcome!**
