# POLO Ontology

## Project Overview

**POLO** (Prebiotic Origins of Life Ontology) is an ontology and RML-based data pipeline for representing and querying experimental prebiotic chemistry data extracted from scientific papers. Developed at Rensselaer Polytechnic Institute.

Base IRI: `https://purl.org/polo`

## File Roles

| File | Purpose |
|------|---------|
| `polo.owl` | OWL ontology defining all classes, object properties, and data properties |
| `mapping.ttl` | RML mapping rules that transform `experiments.csv` into RDF triples |
| `experiments.csv` | Source tabular data; one row = one experiment |
| `output.ttl` | Generated RDF knowledge graph (Turtle format) — produced by running the RML mapper |

## Running the RML Mapper

To regenerate `output.ttl` from `experiments.csv` and `mapping.ttl` using [RMLMapper](https://github.com/RMLio/rmlmapper-java):

```bash
java -jar rmlmapper.jar -m mapping.ttl -o output.ttl -s turtle
```

To validate `polo.owl` with Apache Jena's `riot`:
```bash
riot --validate polo.owl
```

To run a SPARQL query against `output.ttl`:
```bash
arq --data output.ttl --query query.sparql
```

## Ontology Architecture

```
Experiment
├── hasMetadata → PaperMetadata        (title, author, DOI, year, journal)
└── hasSetup → ExperimentSetup         (systemType, durationHrs, replicatesCount)
               └── hasExperimentType → ExperimentType
                                        ├── Batch
                                        │   ├── hasConditions → LoadingConditions
                                        │   │                   (pressure, volume, isRoomTemperature, containerMaterial)
                                        │   ├── hasState → InitialState
                                        │   └── hasState → FinalState
                                        └── Flowthrough (separate column set)

InitialState / FinalState
├── hasConditions → SystemConditions   (pressure, temperature, volume, isIsobaric, isIsothermal, isIsochoric)
├── hasPhase → GasPhase                (gasName, gasConcentration, ambientAtmosphere, gasIsMeasured)
├── hasPhase → LiquidPhase             (solventType, soluteName, soluteConcentration, isPolymer, polymerLength,
│                                       noOfUniqueMonomers, monomerName, freqOfMonomer, liquidIsMeasured)
└── hasPhase → SolidPhase              (solidName, solidMass, solidType, solidIsMeasured, maxGrainSize, minGrainSize)
```

POLO imports the [Units Ontology (UO)](http://purl.obolibrary.org/obo/uo.owl) for measurement units. Implicit units: pressure in bars, temperature in °C, volume in mL, solidMass in mg, soluteConcentration in mM/L, grain size in µm.

## RML Mapping Conventions

- Mapping namespace prefix: `:` → `https://purl.org/polo/mapping#`
- Data/ontology namespace: `polo:` → `https://purl.org/polo#`
- All 15 TriplesMap objects share the same logical source (`experiments.csv`, `ql:CSV`)
- URI templates use the `{id}` column as the row key for all experiment-scoped entities
- Missing/unknown values in the CSV are represented as `--`; boolean flags (e.g. `unknown_replicates_count`, `polymer_length_unknown_final_*`) are used to indicate when a numeric value is absent

## Key Design Decisions

- **Flat CSV, structured RDF**: The CSV is intentionally flat (one wide row per experiment); the mapping file handles decomposition into the nested class hierarchy.
- **Multi-valued properties via numbered columns**: Multiple solutes, gases, and monomers are encoded as numbered column suffixes (`_1`, `_2`, … `_5`). Each maps to the same RDF predicate (e.g. `polo:soluteName`) on the same subject URI, producing multiple triples.
- **Batch-only mapping currently active**: The current `mapping.ttl` maps only `experiment_type = batch` rows. Flowthrough experiment columns exist in the CSV but do not yet have corresponding TriplesMap entries.
