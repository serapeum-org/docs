# Package dependency graph

How the Serapeum Python packages depend on each other. Arrows read **"depends on"** — a package points at what it
needs, so the foundation layers sit at the bottom and the applications at the top.

Labels name the extra on **both** ends of a link:

- **Solid arrows** are required dependencies (`project.dependencies`). A label is the extra the dependency is
  installed with — `digital_earth -->|tiles| cleopatra` means `digitalearth` requires `cleopatra[tiles]`.
- **Dashed arrows** are optional. The label reads `<extra on this package> → <extra requested on the dependency>`,
  and the right-hand side is dropped when the dependency is requested plain. So `hapi -.->|inputs → ecmwf|
  earthlens` means `pip install hapi-nile[inputs]` brings in `earthlens[ecmwf]`.
- A dashed arrow **alongside** a solid one is an extra that *upgrades* a dependency you already have:
  `digital-rivers` always needs `pyramids-gis`, and `digital-rivers[distributed]` swaps it for `pyramids-gis[lazy]`.
- `dev`, `docs` and `notebooks` are PEP 735 dependency groups, not user-installable extras — they affect
  contributors, not consumers. `·` separates several extras that produce the same link.

```mermaid
graph TD
    %% ===== applications / models =====
    earthstudio["earthstudio<br/><i>studio backend</i>"]
    hyd_models["hydrological-models"]
    hapi["Hapi<br/><i>hapi-nile</i>"]
    serapis["Serapis<br/><i>serapis</i>"]

    %% ===== domain libraries =====
    digital_earth["Digital-Earth<br/><i>digitalearth</i>"]
    earthlens["earthlens"]
    geostatista["geostatista"]
    digital_rivers["digital-rivers"]
    pyramids_eo["pyramids-eo"]

    %% ===== engine =====
    pyramids["pyramids<br/><i>pyramids-gis</i>"]
    cleopatra["cleopatra"]

    %% ===== foundation =====
    hpc["hpc<br/><i>hpc-utils</i>"]
    statista["statista"]
    oasis["oasis<br/><i>Oasis-Optimization</i>"]
    serapeum_utils["serapeum_utils"]

    %% ===== outside the stack =====
    serapeum["serapeum<br/><i>LLM framework</i>"]
    unicloud["unicloud<br/><i>standalone</i>"]
    floodsim["floodsim<br/><i>C++/CUDA, standalone</i>"]

    %% ---- required dependencies ----
    pyramids --> hpc
    cleopatra --> hpc
    pyramids_eo --> pyramids
    digital_rivers --> pyramids
    geostatista --> pyramids
    earthlens --> pyramids
    digital_earth --> pyramids
    digital_earth -->|tiles| cleopatra
    hapi --> pyramids
    hapi --> cleopatra
    hapi --> statista
    hapi --> oasis
    serapis --> pyramids
    serapis --> cleopatra
    serapis --> statista
    serapis --> serapeum_utils
    hyd_models --> hapi
    hyd_models --> pyramids

    %% ---- optional: reached only through an extra ----
    pyramids -.->|viz → tiles| cleopatra
    digital_rivers -.->|viz → tiles| cleopatra
    geostatista -.->|viz · dev/docs| cleopatra
    earthlens -.->|dev/docs → tiles| cleopatra
    earthlens -.->|eedai · dev/notebooks| pyramids_eo
    hapi -.->|inputs → ecmwf| earthlens
    hyd_models -.->|inputs → ecmwf| earthlens
    earthstudio -.->|gis| pyramids
    earthstudio -.->|gis| pyramids_eo
    earthstudio -.->|gis| geostatista
    earthstudio -.->|gis| digital_earth
    earthstudio -.->|agent| serapeum

    %% ---- extras that upgrade an already-required dependency ----
    pyramids_eo -.->|viz → viz| pyramids
    digital_rivers -.->|viz → viz · distributed → lazy| pyramids
    geostatista -.->|viz → viz · distributed → lazy| pyramids
    earthlens -.->|stac → stac · nwm → parquet| pyramids

    classDef app fill:#fde8e8,stroke:#b03a3a,color:#4a1414
    classDef lib fill:#e6f0fb,stroke:#3a6ea5,color:#132a45
    classDef engine fill:#fff3d6,stroke:#b8860b,color:#4a3405
    classDef base fill:#e8f4ea,stroke:#4a7c59,color:#1b3a24
    classDef solo fill:#f0eef6,stroke:#6b5b95,color:#2a2240

    class earthstudio,hyd_models,hapi,serapis app
    class digital_earth,earthlens,geostatista,digital_rivers,pyramids_eo lib
    class pyramids,cleopatra engine
    class hpc,statista,oasis,serapeum_utils base
    class serapeum,unicloud,floodsim solo
```

## The layers

| Layer | Packages | Role |
|---|---|---|
| Foundation | `hpc-utils`, `statista`, `Oasis-Optimization`, `serapeum_utils` | Numeric, statistical and optimization primitives with no intra-stack dependencies |
| Engine | `pyramids-gis`, `cleopatra` | The GDAL/OGR layer and the matplotlib layer everything else builds on |
| Domain libraries | `pyramids-eo`, `digital-rivers`, `geostatista`, `digitalearth`, `earthlens` | Specialized capabilities on top of the engine — EO, DEM, geostatistics, visualization, data acquisition |
| Applications | `hapi-nile`, `serapis`, `hydrological-models`, `earthstudio` | End-user models and services |

## Notes

**`cleopatra` is an optional dependency of the engine.** `pyramids-gis`, `digital-rivers` and `geostatista` all
reach it through a `viz` extra rather than a required dependency, so a bare install of any of them pulls no
plotting stack. The convention that plotting goes through `cleopatra` is still binding — it is enforced by code
review, not by the dependency metadata.

**Each layer owns its concern.** `digital-rivers` is the source of truth for DEM processing, `pyramids-gis` for
GIS, and `cleopatra` for matplotlib. Downstream packages are expected to depend upward rather than reimplement a
capability that already has a home.

**`pyramids-eo` never declares GDAL.** `pyramids-gis` vendors it; declaring it again would force a from-source
build on platforms with no wheel.

**Three packages sit outside the graph.** `unicloud` has no edges in either direction. `floodsim` is a C++/CUDA
engine with no Python dependencies. `serapeum` (the LLM framework) depends on nothing in the stack, and is reached
only through `earthstudio`'s `agent` extra.
