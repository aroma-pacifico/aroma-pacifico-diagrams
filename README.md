# aroma-pacifico-diagrams

Diagrams as code for the Aroma Pacífico pilot (Draw.io and Mermaid sources with SVG and PNG exports), including the numbered boxes D01-D13 that the IaC implements

Part of the Aroma Pacífico pilot, the 2026 update of the coffee traceability pilot for Cauca, Huila and Tolima. Work is tracked in [aroma-pacifico-roadmap](https://github.com/aroma-pacifico/aroma-pacifico-roadmap) and on the [Aroma Pacífico Pilot board](https://github.com/orgs/aroma-pacifico/projects/1).

## What lives here

This repository holds diagram sources (uncompressed Draw.io XML and Mermaid files) and their publication exports (editable SVG and preview PNG), covering architecture, data-flow, component interactions, and the numbered infrastructure boxes D01 through D13. Distributed AI agent rules and standards are synchronized here from the canonical `aroma-pacifico-ai-agents` repository.

## Data paths

The A13 data-path package documents crop and drying telemetry, transporter
positions, and drone objects. Each path includes an editable Draw.io source,
an SVG publication export, and a 2400-pixel PNG preview.

```text
data-paths/
|-- data-paths.md
|-- telemetry.drawio
|-- telemetry.svg
|-- telemetry.png
|-- transporter-position.drawio
|-- transporter-position.svg
|-- transporter-position.png
|-- drone-objects.drawio
|-- drone-objects.svg
`-- drone-objects.png
```

## Tasks delivered here

- [A13](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/13) data paths
- [A19](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/19) complete diagram (the document lives in aroma-pacifico-architecture)
- Diagrams requested by the architecture tasks
