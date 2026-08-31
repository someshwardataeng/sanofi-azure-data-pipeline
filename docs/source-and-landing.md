# Sanofi Azure Data Pipeline — Source & Landing Design

## 1. Purpose

This document defines the source-file organization and landing-zone design for the Sanofi Azure Data Pipeline modernization project.

The project recreates the source and landing behavior of the original enterprise IICS/Snowflake solution using Azure services and synthetic data.

> **Important:** All files used in this public project are synthetic. No production, client, Sanofi, IQVIA, credentials, or confidential business data is included.

## 2. Business Context

The source system provides IQVIA-style medical/product data.

Files are received from the client periodically and contain product-level information such as:

- Product Group
- Active Ingredient
- Formulation
- Strength
- Product Code
- Manufacturer
- Pack Size
- Monthly Units
- Monthly Sales

The source data is available across multiple countries and business categories.

### Countries

The project currently simulates:

- Saudi Arabia
- Czech Republic
- Australia
- India

The production environment contained many more countries.

### Business Categories

Each country can receive data for:

- National
- Subnational
- Distribution

### File Frequencies

Each category is further separated into:

- Monthly
- Weekly

### Supported File Types

The source feed can contain:

- CSV
- Excel

## 3. Original Source Architecture

The original project received client files through a NAS / Windows shared-folder structure.

```text
Client
  ↓
NAS / Windows File Share
  ↓
IICS
  ↓
Snowflake STG
```

The NAS was organized by business category and frequency so that files from multiple countries and feeds could be managed independently.

## 4. Azure Modernization — Source & Landing Architecture

For the Azure implementation:

```text
Client
  ↓
NAS / Windows File Share
  ↓
Self-hosted Integration Runtime
  ↓
Azure Data Factory
  ↓
ADLS Gen2 Landing Zone
```

ADLS Gen2 acts as the controlled cloud-side copy of incoming source files.

Downstream processing starts from the landed files rather than depending directly on the client's NAS after ingestion.

## 5. Why ADLS Gen2?

### 5.1 Source Copy / Source of Truth

The landed file provides a copy on our side.

If the client removes or replaces a file from the NAS, the previously received source file remains available in the cloud landing layer.

### 5.2 Decoupling

Once the file is safely landed, downstream processing can work from ADLS without continuously depending on the availability of the client NAS.

### 5.3 Reprocessing

Keeping the source file allows the engineering team to reprocess a file when a downstream transformation or processing issue occurs.

### 5.4 Auditability

The landing zone provides a historical record of files that entered the platform.

### 5.5 Operational Control

Files can be separated into active landing, archive, and failed/review locations, making operational status easier to understand.

## 6. ADLS Gen2 Container

The project uses a dedicated container:

```text
sanofi
```

Development storage account:

```text
stsanofidev001
```

The container acts as the filesystem boundary for the Sanofi project.

## 7. Landing Folder Structure

The landing zone follows the source business organization:

```text
sanofi/
│
├── landing/
│   ├── SaudiArabia/
│   │   ├── National/
│   │   │   ├── Monthly/
│   │   │   │   └── Archive/
│   │   │   └── Weekly/
│   │   │       └── Archive/
│   │   ├── Subnational/
│   │   │   ├── Monthly/
│   │   │   │   └── Archive/
│   │   │   └── Weekly/
│   │   │       └── Archive/
│   │   └── Distribution/
│   │       ├── Monthly/
│   │       │   └── Archive/
│   │       └── Weekly/
│   │           └── Archive/
│   ├── CzechRepublic/
│   ├── Australia/
│   └── India/
│
└── failed/
```

The same `National / Subnational / Distribution` and `Monthly / Weekly` structure is applied to each country.

### Operational behavior

Each active Monthly or Weekly folder contains the currently available files.

After a file is successfully processed through the required downstream load, including completion of the DWH load, the file is moved to that feed's `Archive` folder.

A timestamp is added to the archived file name to distinguish historical deliveries.

## 8. Successful File Flow

```text
Client
  ↓
NAS / Windows File Share
  ↓
ADF picks up file
  ↓
ADLS Gen2 landing
  ↓
Snowflake STG
  ↓
PSA
  ↓
DWH
  ↓
DMT
  ↓
Move source file to Archive
```

The archive movement occurs only after the required DWH processing has completed successfully.

Example:

```text
Before:
National/Monthly/
└── saudi_arabia_national_monthly_aug_2024.csv

After successful DWH completion:
National/Monthly/
└── Archive/
    └── saudi_arabia_national_monthly_aug_2024_20260831_221530.csv
```

## 9. Failed File Flow

A malformed or invalid file is **not moved to Archive**.

For example, if a file contains a column mismatch or another validation issue:

```text
File arrives
    ↓
ADF processes file
    ↓
Validation fails
    ↓
File remains in current location
    ↓
Failure notification is sent
```

This allows the engineering team to inspect the original file and determine the cause of failure.

## 10. File Selection / Parameterization

The ingestion process should not be hardcoded to one specific monthly file name.

Instead, the design uses a file pattern.

Example:

```text
saudi*.csv
```

This allows the pipeline to pick up the appropriate incoming file without creating a new pipeline for every month.

The same design principle can be extended using country, category, frequency, and file-type parameters.

Example:

```text
Country   = SaudiArabia
Category  = National
Frequency = Monthly
Pattern   = saudi*.csv
```

## 11. Why Country / Category / Frequency Folders?

The production environment contained many countries and multiple business feeds.

Organizing the files by:

```text
Country
  ↓
Category
  ↓
Frequency
```

provides:

- Clear organization
- Separation between country feeds
- Separation between National/Subnational/Distribution data
- Separation between monthly and weekly processing
- Easier operational troubleshooting
- Cleaner file management
- Easier future scaling as additional countries are onboarded

Instead of:

```text
landing/
└── all_files/
```

the platform uses:

```text
landing/
├── SaudiArabia/
├── CzechRepublic/
├── Australia/
└── India/
```

with category and frequency underneath.

## 12. Source Data Shape

The incoming source is intentionally modeled in a wide format to reproduce the original project pattern.

Example:

```text
product_group
active_ingredient
formulation
strength
product_code
manufacturer
pack_size

sep_month_2022_units
oct_month_2022_units
...
aug_month_2024_units

sep_month_2022_sales
oct_month_2022_sales
...
aug_month_2024_sales
```

The same pattern is used across National, Subnational, and Distribution feeds.

The source is initially loaded into Snowflake STG in this wide form.

The PSA transformation later converts the monthly measures into a normalized structure.

## 13. Downstream Transformation Context

```text
ADLS Gen2
    ↓
ADF
    ↓
Snowflake STG
    ↓
PSA View
    ↓
UNPIVOT
    ↓
PSA Table
    ↓
DWH
    ├── Dimensions
    └── Facts
    ↓
DMT
    ↓
Power BI
```

The PSA layer converts the wide monthly source into a row-based structure containing:

```text
Dimension attributes
        +
month_year
        +
units
        +
sales
```

The PSA processing also supports MD5-based change detection and `latest_flag` handling, which will be documented separately in the PSA/data-processing design.

## 14. Design Decisions

| Decision | Rationale |
|---|---|
| Use ADLS Gen2 landing | Maintain a cloud-side source copy |
| Preserve country hierarchy | Many countries and feeds |
| Separate business categories | National, Subnational, Distribution have distinct feeds |
| Separate monthly/weekly | Different processing frequencies |
| Archive successful files | Preserve historical deliveries |
| Timestamp archived files | Prevent filename ambiguity |
| Keep failed files in place | Allow inspection and retry |
| Use file patterns | Avoid monthly hardcoding |
| Use synthetic data | Public GitHub repository must not contain confidential data |

## 15. Current Project Scope

This document covers:

- Source organization
- NAS / Windows file-share concept
- Azure landing architecture
- ADLS Gen2 folder design
- Successful file handling
- Failed file handling
- Archive behavior
- File-pattern design
- Source data shape

The following will be documented separately as implemented:

- ADF pipeline architecture
- Self-hosted Integration Runtime configuration
- Snowflake STG
- PSA transformation and UNPIVOT logic
- MD5 / latest-flag processing
- DWH dimension and fact processing
- DMT loading
- Audit framework
- Monitoring and notifications
- Testing and failure scenarios
