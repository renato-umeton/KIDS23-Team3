# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an OMOP CDM (Observational Medical Outcomes Partnership Common Data Model) ETL project for the KIDS23 biohackathon. The project converts Synthea synthetic patient data (specifically for Sickle Cell Disease patients) into OMOP CDM v5.4 format and performs data quality validation using Achilles.

### Key Technologies
- R with OHDSI libraries (DatabaseConnector, ETLSyntheaBuilder, Achilles)
- PostgreSQL database backend
- OMOP CDM v5.4
- Synthea v3.0.0 synthetic patient data

## Database Architecture

### Database Connection Details
All scripts connect to a PostgreSQL database with these standard settings:
- Server: `broadsea-atlasdb/postgres`
- User: `postgres`
- Password: `mypass`
- Port: `5432`

### Schema Organization
- `cdm_synthea10`: Main OMOP CDM schema containing standardized clinical data
- `vocab`: Vocabulary schema for OMOP standardized vocabularies
- `native`: Synthea native format tables (patient, provider, encounters, etc.)
- `synthea_results`: Schema for Achilles results and data quality metrics

### Important Database Tables
**OMOP CDM Tables:**
- Standard vocabulary tables: `CONCEPT`, `vocabulary`, `concept_ancestor`, `concept_class`, `concept_relationship`, `concept_synonym`
- Clinical data: `person`, `provider`, `condition_occurrence`, `drug_exposure`, `procedure_occurrence`, etc.

**Native Synthea Tables:**
- Modified schema with custom fields:
  - `patients`: Added `fips` (numeric), `income` (numeric)
  - `providers`: Dropped `utilization`, Added `encounters` (numeric), `procedures` (numeric)

## Data Pipeline

### 1. Synthea CSV Files
Located in `scd_csv/` directory containing synthetic patient data:
- `patients.csv`, `providers.csv`, `organizations.csv`
- Clinical data: `encounters.csv`, `conditions.csv`, `procedures.csv`, `medications.csv`
- `observations.csv`, `immunizations.csv`, `allergies.csv`, `careplans.csv`
- `devices.csv`, `imaging_studies.csv`, `supplies.csv`

Note: CSV files must be available at `/tmp/KIDS23-Team3/scd_csv` when running ETL scripts.

### 2. ETL Workflow (test_synthea_to_omop.R)
The complete ETL process follows this sequence:

1. **CreateCDMTables**: Create empty OMOP CDM v5.4 tables in target schema
2. **CreateSyntheaTables**: Create Synthea native format tables
3. **LoadSyntheaTables**: Load Synthea CSV files into native schema
4. **LoadVocabFromCsv**: Load OMOP vocabularies from `/tmp/omop_vocab`
5. **LoadEventTables**: Transform and load Synthea data into CDM tables

### 3. Data Quality Validation (run_achilles.R)
Achilles runs 200+ data quality analyses and outputs results to:
- Database: `synthea_results` schema
- Files: `output/` directory containing SQL scripts and logs

## Common Commands

### Running the Complete ETL Pipeline
```r
# Install required package (first time only)
devtools::install_github("OHDSI/ETL-Synthea")

# Run the full ETL
source("test_synthea_to_omop.R")
```

### Running Data Quality Checks
```r
source("run_achilles.R")
```

### Testing Database Connectivity and Data
```r
source("test_data_load.R")
```

### Querying for Sickle Cell Disease Concepts
```r
# Example pattern from test_data_load.R
tbl(conn, in_schema("cdm_synthea10", "CONCEPT")) %>%
  filter(concept_name %like% "Sickle%" | concept_name %like% "sickle%") %>%
  select(concept_id, concept_name, domain_id, vocabulary_id, concept_class_id) %>%
  print(n = 100)
```

## Development Notes

### File Structure
- `test_synthea_to_omop.R`: Main ETL script with complete pipeline
- `run_achilles.R`: Achilles data quality analysis script
- `test_data_load.R`: Database connectivity and exploratory queries
- `scd_csv/`: Source Synthea CSV files for Sickle Cell Disease cohort
- `output/`: Achilles results (SQL scripts, logs, data quality metrics)
- `scd.json`: Synthea disease module configuration for SCD
- `output/synthea-*.json`: Synthea JSON output files

### ETL Customizations
The project includes modifications to standard Synthea schema:
- Patient table has additional socioeconomic fields (fips, income)
- Provider table structure modified to track encounters and procedures

### Prerequisites
The following must be available on the system:
- OMOP vocabulary files at `/tmp/omop_vocab/`
- Synthea CSV files at `/tmp/KIDS23-Team3/scd_csv/`
- PostgreSQL database with appropriate schemas created

### Achilles Output
After running Achilles, the `output/` directory contains:
- `achilles.sql`: Main analysis results
- Multiple validation scripts: `isPrimaryKey.sql`, `isForeignKey.sql`, `isRequired.sql`
- Data quality checks: `plausibleValueHigh.sql`, `plausibleValueLow.sql`, `plausibleGender.sql`, etc.
- Completeness metrics: `measurePersonCompleteness.sql`, `sourceValueCompleteness.sql`
- Log files: `log_achilles.txt`, `log_createIndices.txt`
