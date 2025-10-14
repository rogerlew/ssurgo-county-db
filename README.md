# SSURGO County Database for WEPP Soil Generation

## Overview

This repository contains a comprehensive database of soil files for all U.S. counties, generated for use with the Water Erosion Prediction Project (WEPP) model. The dataset provides county-representative soil profiles derived from USDA's Soil Survey Geographic Database (SSURGO) and State Soil Geographic Database (STATSGO2).

## Purpose

The SSURGO County Database was developed to provide a standardized, readily-accessible collection of soil parameters for WEPP modeling across the United States. By identifying and processing the dominant soil type for each county, this dataset enables:

- Large-scale erosion modeling and soil loss predictions
- Consistent soil parameterization across regional to national scales
- Rapid initialization of WEPP models without manual soil data collection
- Fallback soil data using STATSGO2 where SSURGO data is unavailable or invalid

## Dataset Contents

### Generated Files

The repository contains the following key outputs:

- **`soils/`** - Directory containing 2,923 WEPP soil files (.sol) and corresponding logs (.log)
- **`soil_lookup.csv`** - Master lookup table mapping counties to soil files
- **`dom_mukeys_by_county.csv`** - Dominant SSURGO Map Unit Keys (MUKEYs) for each county
- **`failed_counties.txt`** - Counties where soil generation failed (6 territories)

### Coverage Statistics

- **Total Counties**: 3,227 U.S. counties
- **WEPP Soil Files Generated**: 2,923 unique soil profiles
- **SSURGO-derived**: 2,738 counties (84.8%)
- **STATSGO-derived**: 283 counties (8.8%)
- **Failed/No Data**: 206 counties (6.4%)

### File Formats

#### WEPP Soil Files (*.sol)
WEPP-format soil parameter files containing:
- Soil layer properties (texture, bulk density, organic matter)
- Hydraulic properties (saturated hydraulic conductivity)
- Erodibility parameters (critical shear stress, interrill/rill erodibility)
- Format version: 2006.2ag

#### soil_lookup.csv
Lookup table with the following fields:
- `AFFGEOID` - American FactFinder Geographic Identifier (county FIPS code)
- `MUKEY` - SSURGO/STATSGO Map Unit Key
- `source` - Data source (either "surgo" or "statsgo")
- `lng`, `lat` - County centroid coordinates (WGS84)

## Dataset Construction Methodology

The dataset was constructed through a multi-stage pipeline designed to identify representative soils for each U.S. county:

### Stage 0: Determine Dominant MUKEYs (`_0_determine_dominate_mukeys.py`)

This script identifies the most representative soil type for each county:

1. **Input Data**: U.S. Census Bureau county shapefiles (cb_2017_us_county_500k)
2. **Process**:
   - Iterates through all U.S. counties
   - Retrieves SSURGO raster data (90m resolution) for each county extent
   - Creates a mask for the county polygon
   - Extracts all SSURGO MUKEYs within county boundaries
   - Determines the dominant (most common) MUKEY by pixel count
   - Calculates county centroid coordinates for STATSGO fallback
3. **Outputs**:
   - `dom_mukeys_by_county.csv` - Successful county/MUKEY mappings
   - `failed_counties.txt` - Counties where processing failed

**Key Design Decision**: The dominant MUKEY approach assumes the most spatially extensive soil type best represents the county for regional-scale modeling. This avoids arbitrary selection while maintaining a single representative soil per county.

### Stage 1: Build WEPP Soil Database (`_1_build_database.py`)

This script converts SSURGO/STATSGO data to WEPP soil format:

1. **SSURGO Processing**:
   - Reads dominant MUKEYs from Stage 0
   - Queries USDA SSURGO SOAP server for soil component data
   - Processes 100 MUKEYs at a time to avoid server overload
   - Uses `wepppy.soils.ssurgo.SurgoSoilCollection` to:
     - Retrieve soil horizon data
     - Calculate WEPP soil parameters
     - Validate soil profiles
   - Generates .sol files with WEPP 2006.2ag format

2. **STATSGO Fallback**:
   - For counties where SSURGO data is invalid or unavailable
   - Uses county centroid coordinates to identify STATSGO MUKEY
   - Queries STATSGO database using `StatsgoSpatial.identify_mukey_point()`
   - Generates WEPP soil files from coarser-resolution STATSGO data

3. **Outputs**:
   - `soils/` directory with .sol and .log files
   - `soil_lookup.csv` mapping counties to generated soil files

**Data Source Hierarchy**:
1. SSURGO (preferred) - High-resolution, survey-grade soil data
2. STATSGO2 (fallback) - Generalized soil data at state level
3. None - Territories and areas without coverage

### Stage 2: Quality Validation (`_2_soil_lookup_check.py`)

Final validation to ensure data integrity:

1. Verifies all MUKEYs in lookup table have corresponding .sol files
2. Identifies and removes broken references
3. Generates `soil_lookup_checked.csv` with validated mappings

## Dependencies

The dataset generation process requires:

- **wepppy** - WEPP Python library for soil processing
  - `wepppy.soils.ssurgo.SurgoSoilCollection` - SSURGO/STATSGO data retrieval and WEPP conversion
  - `wepppy.soils.ssurgo.StatsgoSpatial` - Spatial queries for STATSGO
  - `wepppy.all_your_base.geo` - GIS operations (shapefile reading, raster processing, coordinate transformations)
- **numpy** - Numerical operations
- **matplotlib** - Optional visualization

## Usage

### Accessing Soil Data for a County

```python
import csv

# Load the lookup table
with open('soil_lookup.csv') as f:
    reader = csv.DictReader(f)
    lookup = {row['AFFGEOID']: row for row in reader}

# Get soil file for a specific county (e.g., Ada County, Idaho)
fips = '0500000US16001'
if fips in lookup and lookup[fips]['MUKEY'] != 'None':
    mukey = lookup[fips]['MUKEY']
    soil_file = f'soils/{mukey}.sol'
    print(f"Soil file: {soil_file}")
    print(f"Data source: {lookup[fips]['source']}")
```

### Reading WEPP Soil Files

WEPP soil files (.sol) are ASCII text files that can be read directly by WEPP models or parsed using the wepppy library:

```python
from wepppy.wepp.soils import Soil

soil = Soil('soils/328231.sol')
# Access soil properties, horizons, etc.
```

## Known Limitations

1. **Failed Counties** (6 territories): SSURGO data is not available for:
   - Kusilvak Census Area, AK (0500000US02016)
   - American Samoa territories (4 islands)
   - Guam territory (1 island)

2. **Spatial Representation**: Each county is represented by a single dominant soil type, which:
   - May not capture the full soil variability within a county
   - Is most appropriate for regional/county-scale modeling
   - Should not be used for field-scale applications where detailed soil maps are available

3. **STATSGO Fallback**: Counties using STATSGO data (283 total) have:
   - Lower spatial resolution than SSURGO
   - More generalized soil properties
   - Potential for reduced accuracy in WEPP predictions

4. **Data Vintage**:
   - County boundaries: 2017 U.S. Census Bureau
   - SSURGO data: March 2017 snapshot
   - WEPP format: 2006.2ag version

## Data Sources

- **SSURGO** (Soil Survey Geographic Database): USDA Natural Resources Conservation Service
- **STATSGO2** (State Soil Geographic Database): USDA Natural Resources Conservation Service  
- **County Boundaries**: U.S. Census Bureau Cartographic Boundary Files (cb_2017_us_county_500k)

## Processing Scripts

The repository includes the full processing pipeline:

- `_0_determine_dominate_mukeys.py` - Identify dominant soil for each county
- `_1_build_database.py` - Generate WEPP soil files from SSURGO/STATSGO
- `_2_soil_lookup_check.py` - Validate soil file references
- `_3_test_mukey.py` - Testing utility for individual MUKEYs

## Citation

If you use this dataset in your research, please cite:

```
SSURGO County Database for WEPP Soil Generation
Repository: https://github.com/rogerlew/ssurgo-county-db
Data Sources: USDA-NRCS SSURGO/STATSGO2, U.S. Census Bureau
```

And acknowledge the underlying data sources:
- Soil Survey Staff, Natural Resources Conservation Service, United States Department of Agriculture. Soil Survey Geographic (SSURGO) Database. Available online at https://sdmdataaccess.sc.egov.usda.gov
- U.S. Census Bureau. (2017). Cartographic Boundary Files - Shapefile.

## Related Resources

- **WEPP Model**: https://www.ars.usda.gov/pacific-west-area/moscow-id/forest-rangeland-ecosystem-science-center/docs/wepp/
- **WEPPpy**: Python library for WEPP model automation
- **SSURGO Database**: https://www.nrcs.usda.gov/resources/data-and-reports/soil-survey-geographic-database-ssurgo

## License

The processing code in this repository is provided as-is for research purposes. The soil data itself is derived from public domain U.S. government sources (USDA-NRCS).

## Contact

For questions about this dataset or processing methodology, please open an issue in this repository.
