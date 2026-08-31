# Sanofi Azure Data Pipeline - Architecture

## Project Overview

This project is an Azure modernization of a
file-based enterprise data integration architecture.

The original architecture used Informatica IICS
and Snowflake.

This project recreates the architecture using
Azure Data Factory, ADLS Gen2, Databricks/PySpark
and Snowflake.

## Status

🚧 Project initialization

## High-Level Architecture

Source Files
→ ADLS Gen2
→ Azure Data Factory
→ Snowflake STG
→ PSA
→ DWH
→ DMT
→ Power BI
