# Future-proof VATSIM open data v0.1

## Terminology

- **Elemental Volume**
  - lateral border: polygon specified by list of WGS84 latitude, longitude pairs
  - vertical border: lower and upper flight level
- **Sector**
  - one or more **Elemental Volume**s
  - contiguous
  - not overlapping with other sectors
  - smallest possible block of airspace a controller can staff
- **Position**
  - combination of callsign (e.g. EDMM_ALB_CTR) and primary frequency (e.g. 134.150)
  - one position can staff multiple sectors at one point in time
  - one sector can only be staffed by one position at one point in time (see Open Questions for issues with FIS, shared airspace and executive/planner controllers), but
  - one sector can be assigned to different positions at different times
    - i.e. if **Position**s _EDMM_TRU_CTR_ and _EDMM_NDG_CTR_ are online, **Sector** _EDMMWLD_ is staffed by _EDMM_NDG_CTR_. When _EDMM_NDG_CTR_ is closed, _EDMMWLD_ then continues to be staffed by _EDMM_TRU_CTR_.

## Motivation

The VATSIM community provides different tools to users of the network that display maps of current air traffic and ATC staffing ([VAT-Spy](https://github.com/vatsimnetwork/vatspy-data-project), VATprism, [SimAware](https://github.com/vatsimnetwork/simaware-tracon-project), VATSIM Radar, [vatglasses](https://github.com/lennycolton/vatglasses-data) and third party tools like Volanta, etc.), aids for controllers to easily and graphically look up transfer agreements (i.e. [ATCISS by the German VACC](https://github.com/vatger/atciss)), flow management/sector load tools (prototypes in Austrian VACC). These all require data of **Sector**s and **Position**s to compute the information they provide accurately.

This document tries to combine the strengths of existing formats while providing a migration path of already collected data. To not increase the burden of teams maintaining the data, tools for (lossy) backward-compatible generation to existing formats shall be provided.

The data is being collected under an **open & permissive** licence in a central, public repository.

### Use Cases

These following are examples of uses of the data provided by this specification. Note that these only use the provided data but some data they might additionally need is not in the scope of this document.

- Map display
  - staffed sectors
  - split visualisation (lateral and vertical)
  - top-down responsibility
- Structured definition of Letters of Agreements (LoA)
- Flow management/sector occupation tooling
  - needs exact sector dimensions
  - position priorities for sectors
- Staffing templates/schedules for events/bookings
  - discovery of available position
  - "map timeline of bookings"
- Generation of sector data used by ATC clients
- AfV stations definitions
- Check staffing restrictions for Tier 1/2, TVCP airspace groups
- NAV/OPS VACC staff tooling (UI to edit this data)
- "single source of truth" for most of this data
  - enables VACCs to easily update data like CTAF frequency

### Shortcomings of current data formats

**TODO** flesh out, wording

- No vertical borders, sectors can overlap
  - impossible to accurately identify a sector an aircraft is/will be inside
- Missing unique IDs for sectors & elemental volumes
  - cannot be referred to from other data uniquely
    (e.g.: agreement ades: EDDF, cop: DEBHI, level: 240, from: **EDMMALB1** to: **EDGGDKB2**)
- Missing FIR/UIR relation
  - Vatglassses for example only divides data into countries, which is enough for mapping purposes, but not further data processing
- Unclear definition of sector being active if multiple runways are necessary to be active (and vs or)
  - vatglasses provides this data but does not allow the type of logical combinations of runways (_and_ vs _or_)
- Priority of Positions for Sectors only definable in vatglasses
  - more complex splits cannot be displayed on map
  - determination of responsible controller for load computation or LoA application impossible
- Position matching on callsign instead of the actually unique (callsign prefix + callsign suffix + primary frequency)
  - shows EDMM_AXB_CTR as whole FIR being staffed in various existing tools, when mentoring EDMM_ALB_CTR or as callsign used when relieving

## Detailed Design

**TODO** primary goals

- globally contiguous
- non-ambiguity guarantees
- referenceable
- open

### Licence

The code and data in the VATSIM open data repositories shall be licenced as:

- Code: MIT, Apache v2 dual licence ([for reasoning see the Rust discussion](https://internals.rust-lang.org/t/rationale-of-apache-dual-licensing/8952))
- Data: CC-BY-SA-4.0 license (same as VAT-Spy data)

### File Formats

All source data files shall be either of the formats [JSON5](https://spec.json5.org/) or [GeoJSON](https://datatracker.ietf.org/doc/html/rfc7946).

JSON5 files shall be used for the general definition of data, since in comparison with JSON it can contain comments and in comparison with YAML it has less ambiguity in data definition. YAML for example the "norway problem": `de` vs `"de"` => both strings `de`, `no` vs `"no"` => the former boolean `false`, the latter the string `"no"`, [see this post](https://ruudvanasseldonk.com/2023/01/11/the-yaml-document-from-hell).

For solely geographic data GeoJSON may be used, to facilitate modifying the data with external tools. For each item of geographic data two fields shall be provided, one specifying the data inline, the other referencing a `.geojson` file. These are mutually exclusive options.

The files shall be encoded as `UTF-8` and use Unix line endings `\n`.

Generated files (pipeline artifacts) may use other file formats.

### File Structure

```
├──data/
│  ├──EDMM
│  │  ├── elemental_volumes.json5
│  │  ├── elemental_volumes.geojson
│  │  ├── sectors.json5
│  │  ├── positions.json5
│  │  └── airports.json5
│  │  └── airports.geojson
│  ├──LOVV
│  │  ├── elemental_volumes.json5
│  │  ├── elemental_volumes.geojson
│  │  ├── sectors.json5
│  │  ├── positions.json5
│  │  ├── airports.json5
│  │  └── airports.geojson
│  └──...*all FIRs*
├──README.md
└──SPECIFICATION.md (this document)
```

#### `elemental_volumes.json5`

Shall contain all Elemental Volumes in the format defined below as a JSON5 object defined by `key` as key and the other attributes as value.

#### `elemental_volumes.geojson`

Shall contain geographic data for Elemental Volumes as a GeoJSON _FeatureCollection_. Each _Feature_ shall be defined by a _Polygon_ geometry and contain at least a property `id` that allows mapping to the `key` in `elemental_volumes.json5`.

#### `sectors.json5`

Shall contain all Sectors in the format defined below as a JSON5 object defined by `key` as key and the other attributes as value.

#### `positions.json5`

Shall contain all Positions in the format defined below as a JSON5 object defined by `key` as key and the other attributes as value.

#### `airports.json5`

Shall contain all Airports in the format defined below as a JSON5 object defined by `key` as key and the other attributes as value.

#### `airports.geojson`

Shall contain geographic data for Airports as a GeoJSON _FeatureCollection_. Each _Feature_ shall be defined by a _Point_ geometry and contain at least a property `id` that allows mapping to the `key` in `airports.json5`.

### Data Types

#### Coordinate

Tuple of WGS84 decimal coordinates ordered longitude (E positive, W negative), latitude (N positive, S negative), following the GeoJSON standard of defining *Position*s.
_Note: this is the reversed order of most data currently used in VATSIM contexts._

Example: location near Munich, Germany: `[11.5, 48.5]`, in ICAO waypoint format this would be `N4830E01130`.

#### Elemental Volume

##### Constraints

```
lower_level < upper_level
lower_level >= 0
upper_level <= 999
```

##### `key`

- unique identifier within FIR
- type: _string_
- example: identifier from AIP `"EDMMNDG1"`

##### `lower_level`

- the flight level of the lower vertical border (inclusive)
- type: _int_
- example: `195`

##### `upper_level`

- the flight level of the upper vertical border (exclusive)
- type: _int_
- example: `315`

#### Sector

##### Constraints

Sectors "existing" at the same point in time shall be contiguous and not overlap. This means that with the current filtering possibilities in place, only *runway_filter*ed sectors may overlap and these only if they are mutually exclusive.

##### `key`

- identifier, unique (within FIR)
- type: _string_
- example: `"EDMMNDG"`

##### `description`

- human readable name
- type: _string_
- example: `"Nördlingen"`

##### `volumes`

- **Sector** consists of these **Elemental Volumes**
- type: _list of elemental_volume.id_
- example: `"EDMMNDG1", "EDMMNDG2"`

##### `position_priority`

- List of positions specifying the descending priority for sector staffing.
- In case the Position belongs to another FIR, it shall be specfied, otherwise it may be omitted.
- unit _list of references to Positions of the form `{ fir: null or fir_id, id: position_id }`_
- example: `[{ fir: null, id: "DON" }, { fir: "EDMM", id: "ALB" }]`

##### `runway_filter`

- Optional filter to only activate sector on specific runway configurations
- type: _null or list of list of runway_ where _runway_: _`{ airport: ICAO, runway: rwy_designator }`_. The inner list is a logical **and** aggregation, the outer logical **or**.
- Example: `[[{ airport: "ZZZZ", runway: "24L" }, { airport: "ZZZZ", runway: "24R" }], [{ airport: "ZZZZ", runway: "21" }]]` will be active if either (24L and 24R) or 21 of airport `ZZZZ` are active

#### Position

##### `key`

- identifier, unique (within FIR)
- type: _string_
- example: "RDG"

##### `frequency`

- **TODO**: logic for non-available frequency
- type: _null or integer in Hz_
- example: `132555000` for 132.555 MHz

##### `prefix`

- Prefix
- type: _string_
- example: `"EDMM"`

##### `station_type`

- Facility of the position, the suffix in the VATSIM controller callsign.
- type: _one of "FSS", "CTR", "APP", "DEP", "TWR", "RMP", "GND", "DEL", "RDO", "FIS", "TMU"_
- example: `"CTR"` for EDMM_RDG_CTR

##### `name`

- Name of position
- type: _string_
- example: `"Roding"`

##### `radio_callsign`

- Radio callsign of position
- type: _string_
- example: `"München Radar"`

##### `gcap_tier`

- **TODO** tier 1/2 (specify tier 2 further?)
- unit: _null or 1 or 2_

##### `cpdlc_logon`

- Logon code used for CPDLC, globally unique
- type: _null or string_
- example: `"EDMZ"`

##### `airspace_groups`

- Airspace groups according to TVCP section 7.1(c) the position is part of
- type: _null or list of string_

#### Airport

##### `key`

- ICAO code
- type: _string_
- example: `"LIPB"`

##### `name`

- Name of airport
- type: _string_

##### `callsign`

- Radio Callsign of airport, if different from `name`
- unit: _null string_

##### `fallback_prefixes`

- Other prefixes used for matching staffing of TWR or below, should generally not be necessary. They shall be globally unique.
- type: _null or list of string_
- example: `["TOR", "YYZ"]`

##### `topdown_priority`

- **TODO** only necessary for topdown coverage computation
- type: _list of strings of format `position.id` or `"${FIR}/${position.id}"`_
- example: `["LSAS/ARFA", "FUE"]`

##### `runway_configuration`

- Possible runway configurations. This data shall be provided if a Sector has a `runway_filter` depending on this airport, it may be still provided otherwise.
- type: _null or list of list of runways_

### Generation of derived data

#### vatspy

#### simaware

#### vatglasses

#### FIR boundaries

geojson boundaries?

#### .ese sectors and positions

leave out for now?

#### JSON data

**TODO**
global
per fir

### Migration of existing data

Due to licensing automatic migration of data from `vatspy-data-project` is possible. `vatglasses-data` and `simaware-tracon-project` automatic migration is not possible, tooling can be provided but copyright holders of data must be respected!

#### vatspy-data-project

- split `Boundaries.geojson` polygons into non-overlapping polygons retaining mapping. These polygons are the **Elemental Volume**s enriched with `lower_level: 0` and `upper_level: 999`.
  Congruent polygons are deduplicated in the process
- **Airport**s are mapped from the `Airports` section to the format defined herein, whereby IATA codes, FAA LIDs and pseudo airports are combined using `fallback_prefixes`
- `FIRs` and `UIRs` are combined into **Position**s and **Sector**s
  - a CTR **Position** with frequency `null` and prefix `CALLSIGN PREFIX` is created for each entry
  - **Sector**s are created from the relevant combinations of **Element Volume**s and `position_priority` generated (smaller original polygon -> higher priority)

**TODO**: better wording of the above, check for edge cases

#### simaware-tracon-project

No licence exists so all data is copyrighted by the respective author, approval to relicense is required!

Essentially the same process as vatspy-data-project

- "tracons" cutting "holes" into the outerlying sectors, if necessary splitting volume that would contain holes. FL000-999
- create APP **Position**s without frequency and airport prefix
- create **Sector**s with the new "tracon" elemental volumes and priority with the new **Position** and the priority of the outerlying sector

**TODO**: better wording of the above, check for edge cases

#### vatglasses-data

Licence incompatible. Ask lenny colton for relicensing into this project.

**TODO**: mainly "valid" subset of this data definition, so easier to migrate

#### AIXM data

**TODO** Some open data available, maybe provide tooling to convert functional volume and sector data conversions?

### Tooling

Prior to contributing data, it shall be checked for consistency and
adherence to the specification. This may be achieved by running tools in a GitHub Actions pipeline or equivalent alternatives prior to merging.

#### Consistency/Conformity checks

- all coordinates in the GeoJSON files are lists of two items, the first specifying longitude, the second latitude:
  - longitude: -180.0 <= longitude <= 180.0
  - latitude: -90.0 <= latitude <= 90.0
- all polygons defining Elemental Volumes shall not include holes
- Elemental Volume `lower_level` and `upper_level` shall be in the range 0 and 999 inclusive and `lower_level < upper_level`
- Sector `volumes` shall not overlap with `volumes` of other **Sector**s and be continuous (no holes).
- all data references to other data specified herein, shall unambiguously point to valid data:
  - Sector `volumes` shall point to valid Elemental Volumes defined in the same FIR
  - Sector `position_priority` shall point to either a valid Position in the same FIR if none is explicitly specified, otherwise to a valid Position in the specified FIR
  - Sector `runway_filter` shall point to airports defined in the same FIR
  - Sector `runway_filter` runway combinations shall only consist of valid `runway_configuration`s for each Airport
  - The combination of Position `frequency`, `prefix`, `station_type` shall be unique (TODO: how to handle no-frequency case)
  - Position `station_type` shall be checked
  - Position `frequency` shall be an integer in Hz in the HF, VHF or UHF aviation radio communication ranges.
  - Airport `fallback_prefixes` shall be globally unique.

**TODO** most probably incomplete

### Reference implementations

**TODO** planning on implementing "blessed" reference libraries with the following technologies

#### Python + pydantic

#### Rust + serde

#### TypeScript

## Open Questions

**TODO** mainly braindump right now, of things still to check:

- airport.coordinate: Should be ARP?
- replace elemental_volume.json5 by just elemental_volumes.geojson?
- allow inline coordinate/polygon definitions?
- combine geojsons into one file per fir?
- elemental_volume.lateral_border, DME arc support or quantised to polygon? the former would require geojson extension (topojson?). check support in gis tooling
- FIS definition/display (this obviously overlaps but is only relevant to VFR in controlled airspace)
  probably extra files, since the sectors are not necessarily the same geographically (at least not the case in germany), and positions serve other purposes than ATCOs
- CTAF frequency added to airport (initially optional, US only, if rolled out generally, mandatory)
- FIR vs UIR
- allow *null*able keys to be omitted?
- allow other keys?
- allow polygon holes?
- globally unique `fallback_prefixes` vs TWR/GND/DEL as **Position**s
- runway_filter to airports in other filter?
- Position prefix in vatglasses is list of string, is that useful?

## Drawbacks

- Computing sector staffing from a priority as a list of **Position**s on a **Sector** is not able to reflect procedures in the real world, where **Sector**s can be assigned to **Position**s dynamically. For VATSIM this would require a dynamic service aggregating information provided by radar clients and is therefore out of scope of this document defining a static definition format. This can still be added at a later date, with e.g. consumers of this data prefering the dynamic data and falling back to this in case it is not available.

**TODO**

## Alternatives

### TOML instead of JSON5

TOML and JSON5 are very similar regarding their features, where TOML being more verbose defining deeply nested data.

Tooling for TOML seems to be a bit more mature, in case issues arise, the data that can be defined in both formats is equivalent and can easily be moved to TOML

**TODO**

## Future Work

**TODO** flesh out/clean up

LoA
flow management/sector load
airspace volumes (CTRs/airspace A/B/C/D/E/F/G)
navaids/fixes/airways
event schedules/"wanted" positions?
