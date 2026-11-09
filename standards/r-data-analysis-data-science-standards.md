# R Data Analysis & Data Science Standards

## 1. Purpose

These standards define the default practices for **data analysis and data science projects in R**.

The goals are to:

-   Produce analyses that are correct, reproducible, and reviewable.
-   Preserve the provenance of source data and analytical outputs.
-   Make assumptions, transformations, and data-quality decisions explicit.
-   Use established and maintainable R packages where possible.
-   Prevent silent data-quality problems from propagating into analytical results.
-   Make analytical workflows understandable to developers, data scientists, analysts, reviewers, and AI coding agents.
-   Avoid unnecessary complexity and infrastructure.

These standards apply to R-based exploratory analysis, scientific analysis, statistical analysis, modeling, data processing, and analytical deliverables.

General R development conventions (formatting, linting, testing, project structure, dependencies, CI and AI-assisted development) are defined in the R Coding Standards. Standards for data analysis and data science projects in Python are maintained separately.

The level of rigor **SHOULD** be proportional to the importance, complexity, and expected lifetime of the analysis.

These standards apply to new R projects. Existing projects **MAY** adopt them incrementally.

------------------------------------------------------------------------

## 2. How to Apply These Standards

These standards apply equally to work performed by developers, data scientists, analysts, and AI coding agents.

When implementing or modifying an analysis:

1.  Follow existing project conventions when they are consistent with these standards.
2.  Prefer the simplest established tool that satisfies the analytical requirement.
3.  **AVOID** introducing a new dependency when an existing project dependency can reasonably perform the task.
4.  **NEVER** change analytical assumptions, thresholds, units, filtering criteria, or data-quality rules unless explicitly required.
5.  **NEVER** silently correct, discard, impute, interpolate, aggregate, or otherwise alter data.
6.  Preserve source data and maintain a traceable path from source data to analytical outputs.
7.  Inspect and understand relevant data before choosing transformations or cleaning procedures.
8.  Treat unexpected data conditions as information to investigate, not problems to automatically repair.
9.  Preserve existing behavior when modifying an established analysis unless the requested change requires otherwise.
10. **AVOID** unrelated refactoring or modernization while performing an analytical change.
11. Add or update validation and tests when analytical behavior changes.
12. Document decisions that materially affect interpretation or reproducibility.

When requirements are ambiguous and different reasonable choices could materially affect analytical results, **NEVER** guess. Identify the ambiguity and require an explicit analytical decision.

When an unexpected condition indicates a possible data-quality or scientific issue, surface the issue rather than changing the data solely to allow the analysis to complete.

Unexpected data is not necessarily bad data.

------------------------------------------------------------------------

## 3. Requirement Terminology

These terms are capitalized throughout this document and the R Coding Standards:

-   **MUST** / **NEVER**: Required / prohibited.
-   **SHOULD** / **AVOID**: Expected unless there is a documented reason not to.
-   **MAY** / **CONSIDER**: Optional.

Plain instructions without one of these terms (e.g. "Preserve source data") have **SHOULD** strength.

These terms are intended to reduce ambiguity for both human contributors and automated coding agents.

------------------------------------------------------------------------

## 4. Guiding Principles

### Prefer Established Tools

Projects **SHOULD** prefer mature, widely adopted packages with strong ecosystem support.

Long-term stability, documentation, interoperability, developer familiarity, and adoption within the R community (CRAN, Bioconductor, rOpenSci and the tidyverse/r-lib ecosystems) **SHOULD** be considered when selecting dependencies.

Newer or specialized packages **SHOULD** be introduced only when they provide a clear benefit over established tools or straightforward base R implementations.

### Prefer Simple Solutions

Use the simplest tool appropriate for the problem.

A normal R function is often preferable to introducing a framework.

Additional abstraction is appropriate when it provides meaningful benefits in:

-   Reuse
-   Consistency
-   Reliability
-   Maintainability
-   Observability
-   Scale

### Make Analytical Decisions Explicit

Analytical code **MUST** make decisions that materially affect analytical results visible and reproducible.

Code **NEVER** silently:

-   Drops observations.
-   Imputes missing values.
-   Removes outliers.
-   Converts units.
-   Converts or localizes time zones.
-   Aggregates data.
-   Interpolates values.
-   Extrapolates values.
-   Changes coordinate reference systems.
-   Coerces invalid values.
-   Excludes data that fails quality checks.

When these operations are necessary, they **MUST** be intentional and understandable from the code.

Several common R behaviors perform these operations without obvious signals. In particular:

-   `na.rm = TRUE` in summary functions silently excludes missing values.
-   `inner_join()` silently drops rows without a match. `left_join()` can silently duplicate rows when keys are not unique.
-   `as.numeric()`, `as.integer()` and `as.Date()` convert invalid values to `NA`, with at most a warning.
-   `readr` and `readxl` guess column types and convert values that don't fit the guessed type to `NA`.
-   `lubridate` and base date-time functions assume UTC or the system time zone when no time zone is given.
-   Model functions such as `lm()` and `glm()` drop incomplete rows by default (`na.action = na.omit`).

Code using these functions **MUST** do so deliberately, and **SHOULD** check or report the number of affected rows where the effect could be material.

### Preserve Reproducibility

Analytical results **SHOULD** be reproducible from their source data, code, configuration, and documented assumptions.

**AVOID** manual modification of intermediate or final analytical datasets.

For important or recurring analyses, reproducibility **NEVER** depends on undocumented manual steps.

Analyses **SHOULD** record their package versions with renv. Packages, and analyses developed within a package structure, do not need to use renv (see R Coding Standards, Section 3).

Analyses that use randomness (simulation, bootstrapping, resampling, Bayesian sampling, machine learning, cross-validation) **MUST** set a seed.

------------------------------------------------------------------------

## 5. Default Data Science Toolchain

General-purpose packages (data manipulation, file reading, statistical modeling, machine learning, visualization, spatial data, dates, pipelines, testing and reporting) are listed in the R Coding Standards, Sections 16 and 24. This section lists only analysis-specific additions.

Use established packages by default and introduce specialized tools only when they provide a clear advantage.

| Purpose | Preferred Tool | Maturity | Guidance |
|------------------|------------------|------------------|------------------|
| Labeled multidimensional data | stars; terra | Established | Use for gridded and spatiotemporal arrays (e.g. NetCDF) |
| Interactive visualization | plotly; leaflet | Established | Use when interactive visualization or maps provide value |
| Data frame validation | Native dplyr / base R; checkmate | Foundational / native | Default for data frame validation and QC |
| Structured validation reports | pointblank | *In development* | *Under review; see Section 9* |
| Storage schemas | arrow | *In development* | *Under review; see Section 9* |
| Production data-quality monitoring | pointblank (agents, informants); validate | *In development* | *Under review; see Section 9* |

These are defaults rather than absolute requirements when project constraints provide a good reason to choose another tool.

### Package Selection

When selecting a new dependency, **CONSIDER**:

1.  Maturity and project history.
2.  Adoption within the R and data-science ecosystem.
3.  Active maintenance, including whether the package is passing CRAN checks.
4.  Documentation quality.
5.  Interoperability with established tools.
6.  Long-term maintenance risk.
7.  Whether the functionality already exists in base R or current project dependencies.
8.  Whether the dependency materially simplifies the implementation.
9.  Whether it requires system libraries (e.g. GDAL, GEOS and PROJ for sf and terra) that complicate installation or deployment.

**AVOID** introducing multiple packages that solve substantially the same problem without a clear reason.

Existing project dependencies **SHOULD** be preferred when they adequately solve the problem.

------------------------------------------------------------------------

## 6. Data Workflow

Analytical workflows **SHOULD** conceptually follow a forward-moving pipeline:

``` text
Source Data
    ↓
Ingestion
    ↓
Validation / QC
    ↓
Cleaning / Standardization
    ↓
Processed / Derived Data
    ↓
Analysis / Modeling
    ↓
Validation
    ↓
Figures / Tables / Deliverables
```

Not every project needs to physically materialize every stage.

The important principle is that transformations have a clear direction and analytical outputs remain traceable to their inputs.

**AVOID** workflows where generated data is manually modified and subsequently treated as source data.

### targets Pipelines

Multi-step analyses that run repeatedly (e.g. annual model runs, recurring reports, analyses rerun as new data arrives) **SHOULD** be implemented as targets pipelines.

targets tracks which steps depend on which code and data, reruns only the steps affected by a change, and records how each output was produced.

A targets pipeline **SHOULD** be structured as follows:

-   Define the pipeline in `_targets.R` at the project root, with one target per meaningful stage (e.g. read, validate, clean, fit, summarize, report).
-   Write each step as a function in `R/`, so it can be tested with testthat. `_targets.R` **SHOULD** only call these functions.
-   Track source data files as file targets (`format = "file"`) so that changes to source data trigger reruns.
-   Render reports from the pipeline with `tarchetypes::tar_quarto()`, so they update when their inputs change.
-   Run the pipeline with `targets::tar_make()`. Use `targets::tar_visnetwork()` to view the dependency graph and `targets::tar_meta()` to inspect errors and timing.
-   Provide run-time settings (e.g. sites or years) through a configuration file or `tarchetypes::tar_map()`, rather than editing pipeline code for each run.
-   **NEVER** commit the `_targets/` data store. Add it to `.gitignore`.

targets is not required for one-off analyses, short scripts, or work where every step runs quickly. **CONSIDER** a plain script or Quarto document in those cases.

------------------------------------------------------------------------

## 7. Source Data and Provenance

### Source Data Is Immutable

Original source data **MUST** be treated as immutable.

Analysis code **NEVER** overwrites source datasets.

If source data requires correction, the original data **MUST** be preserved and the correction **SHOULD** occur as an explicit transformation.

When projects maintain data locally, a structure such as the following is recommended:

``` text
data-raw/       # source data, never modified
data/           # processed and derived data
outputs/        # figures, tables and deliverables (or results/)
```

Projects **MAY** name the deliverables folder `outputs/` or `results/`, used consistently within the project. The exact directory structure **MAY** vary depending on the project and storage system.

Refer to data files relative to the project root with `here::here()`, **NEVER** with absolute paths or `setwd()`.

### Preserve Provenance

Derived datasets **SHOULD** be traceable to:

-   Source dataset or datasets.
-   Code used to generate them.
-   Important parameters.
-   Relevant assumptions.
-   Important filtering or exclusion criteria.

For important or long-lived analytical products, sufficient information **MUST** exist to determine how an output was produced.

In targets pipelines, the pipeline definition and metadata (`tar_meta()`) provide much of this record. Outside targets, the script that produces each derived dataset **SHOULD** be identifiable from the README or the script's own header.

### Generated Data

**AVOID** manually editing generated analytical datasets.

If a value needs to be corrected, the correction **SHOULD** occur in the code or documented input data so that the output can be regenerated.

Manual corrections to source data **SHOULD** be recorded in a version-controlled corrections file (e.g. `data-raw/corrections.csv`, listing record ID, field, original value, corrected value and reason) and applied in code.

------------------------------------------------------------------------

## 8. Data Loading

Choose the data-access method appropriate to the size and structure of the dataset.

### Tabular Data

Column types **SHOULD** be specified explicitly when reading important data, rather than relying on type guessing:

``` r
trap_data <- readr::read_csv(
  here::here("data-raw", "rbdd_catch.csv"),
  col_types = readr::cols(
    date = readr::col_date(),
    site = readr::col_character(),
    count = readr::col_integer(),
    fork_length_mm = readr::col_double()
  )
)
```

**NEVER** ignore parsing problems. Check `readr::problems()` after reading, or fail on them.

Use dplyr and tibbles when:

-   Compatibility with the broader R ecosystem is important.
-   Existing project code is already dplyr-based.
-   Readability for reviewers is a priority.
-   Dataset size and performance do not justify another tool.

Use data.table when:

-   Processing large datasets in memory.
-   Performing substantial transformations or aggregations where performance matters.
-   Performance or memory usage is important.

**AVOID** rewriting working dplyr code into data.table solely for consistency, or the reverse.

Likewise, **AVOID** moving repeatedly between dplyr and data.table without a clear reason.

### Large Analytical Datasets

*In development: guidance on arrow and storage solutions for large datasets is under review.*

**CONSIDER** duckdb or arrow when analysis primarily involves:

-   Filtering.
-   Joining.
-   Aggregating.
-   Querying multiple files.
-   Working directly with Parquet.
-   Datasets larger than convenient in-memory data frame workflows.

Both can be used with dplyr syntax (`arrow::open_dataset()`, or dbplyr with duckdb), which keeps code readable while processing data outside memory. Call `collect()` only after filtering and selecting.

**AVOID** loading an entire dataset into memory when only a subset of rows or columns is required.

Column selection and filtering **SHOULD** be used when supported by the storage format and processing engine (e.g. `col_select` in readr, or `select()` and `filter()` before `collect()` with arrow and duckdb).

### Scientific and Spatial Data

Prefer:

-   stars or terra for labeled multidimensional and gridded scientific data.
-   sf for vector geospatial data.
-   terra for raster data.
-   arrow for Arrow and Parquet interoperability (*in development*).

Projects **SHOULD** choose data structures that represent the underlying problem naturally rather than forcing every dataset into a data frame.

Coordinate reference systems **MUST** be checked on read (`sf::st_crs()`, `terra::crs()`), and transformations **MUST** be explicit (`sf::st_transform()`, `terra::project()`).

------------------------------------------------------------------------

## 9. Data Validation and Quality Control

Analytical inputs **SHOULD** be validated before they are used.

The depth of validation **SHOULD** be proportional to the consequences of incorrect data.

Validation **SHOULD** occur close to important data boundaries, particularly when data:

-   Enters an analysis.
-   Comes from an external system.
-   Is produced by an automated pipeline.
-   Is reused by multiple analyses.
-   Is used to produce important analytical deliverables.

### Data Frame Validation

For analytical workflows, prefer explicit validation using the project's existing data-processing packages.

dplyr, base R or checkmate **MAY** be used directly to check:

-   Required columns.
-   Data types.
-   Null (`NA`) values.
-   Duplicate records.
-   Expected ranges.
-   Allowed categorical values.
-   Unique identifiers.
-   Temporal coverage.
-   Expected row counts or dimensions when relevant.

For example:

``` r
validate_trap_data <- function(df) {
  checkmate::assert_names(
    names(df),
    must.include = c("date", "site", "count", "fork_length_mm")
  )
  checkmate::assert_integerish(df$count, lower = 0)
  checkmate::assert_subset(unique(df$site), ALLOWED_SITES)

  n_dupes <- sum(duplicated(df[c("date", "site")]))
  if (n_dupes > 0) {
    cli::cli_abort("{n_dupes} duplicate date/site record{?s} found.")
  }

  invisible(df)
}
```

Validation logic that is reused across an analysis or project **SHOULD** be implemented as reusable functions rather than duplicated throughout Quarto documents or scripts.

### Joins

Joins **SHOULD** declare their expected relationship so that unexpected duplicates raise errors, e.g. `dplyr::left_join(x, y, by = "site_id", relationship = "many-to-one")`. When rows must not be dropped by an `inner_join()`, use `unmatched = "error"`.

When a join does not declare its relationship, **NEVER** ignore the warnings it raises (e.g. many-to-many relationships). How each warning was investigated and handled **MUST** be documented in a code comment.

**AVOID** introducing a dedicated validation framework when straightforward validation code adequately solves the problem.

### Storage Schemas

*Section in development*

arrow schemas **MAY** be used to define structural expectations for Arrow and Parquet datasets.

Storage schemas are useful for validating:

-   Column names.
-   Data types.
-   Nullability.
-   Nested structures.

Storage schemas are **NEVER** a replacement for scientific or analytical quality-control checks.

### Production Data-Quality Systems

*Section in development*

Dedicated tools such as pointblank or validate **MAY** be appropriate when a project requires:

-   Automated recurring validation.
-   Persistent validation results.
-   Data-quality monitoring.
-   Alerting.
-   Validation across many datasets or pipelines.
-   Operational visibility into data-quality failures.
-   Validation reports shared with clients or reviewers.

**AVOID** introducing a data-quality framework when straightforward validation code adequately solves the problem.

### Validation Failures

**NEVER** silently correct data that fails an important validation check.

Unexpected validation failures **MUST** either:

1.  Stop the analysis (`cli::cli_abort()`), or
2.  Be explicitly handled and documented.

A successful file read does not imply that the data is valid.

**NEVER** automatically assume unexpected values are erroneous.

Warnings raised while reading or transforming data (parsing failures, values coerced to `NA`, many-to-many joins) **SHOULD** be treated as validation failures and investigated. **AVOID** suppressing them. If a warning is suppressed, the code **MUST** include a comment documenting what the warning is and why it is safe to suppress (see R Coding Standards, Section 14).

------------------------------------------------------------------------

## 10. Missing Data

Missing data **MUST** be handled intentionally.

**NEVER** automatically drop observations containing missing values without an analytical reason.

Operations equivalent to:

``` r
tidyr::drop_na(df)
```

``` r
na.omit(df)
```

``` r
dplyr::filter(df, !is.na(fork_length_mm))
```

or summaries using:

``` r
mean(df$fork_length_mm, na.rm = TRUE)
```

are **NEVER** used without understanding why the values are missing and how removing them affects the analysis.

When rows are dropped for missing values, code **SHOULD** report how many were removed (e.g. with `cli::cli_inform()`).

Model functions drop incomplete rows by default. When fitting models on data with missing values, check how many rows were used (`nobs()`) against the number expected.

Depending on the analysis, missing values may represent:

-   Missing observations.
-   Sensor failures.
-   Values below detection limits.
-   Values that are not applicable.
-   Data that failed QC.
-   Gaps in collection.
-   Failed joins.

**NEVER** automatically treat these cases as equivalent.

Imputation **MUST** be intentional and **SHOULD** be documented when it can materially affect analytical results.

R has a single `NA` value per type, so it cannot record why a value is missing. When the reason for missingness is known and analytically relevant, that information **SHOULD** be preserved in a separate column (e.g. a `qc_flag` or `missing_reason` column) rather than representing all missing-data states as a single `NA`.

**NEVER** replace missing values with zero (e.g. with `tidyr::replace_na()` or `dplyr::coalesce()`) unless zero is analytically correct for that variable and context. In particular, a missing catch count is not the same as zero catch.

------------------------------------------------------------------------

## 11. Automation and AI Agent Safeguards

The general rules for AI-assisted development in the R Coding Standards, Section 25, apply to all analytical work. This section adds safeguards specific to analysis.

Automated tools and AI coding agents **MUST** follow the same analytical and scientific standards as human contributors.

Automation **NEVER** independently changes scientific or analytical intent.

In particular, automated tools and AI agents **NEVER**:

-   Invent missing data.
-   Assume unexpected values are erroneous.
-   Remove observations merely because they appear unusual.
-   Change thresholds or filtering criteria without instruction.
-   Select an imputation strategy without analytical justification.
-   Infer units when units are ambiguous.
-   Assume a timezone when timestamps are ambiguous.
-   Convert units without making the conversion explicit.
-   Change coordinate reference systems without making the transformation explicit.
-   Change statistical or modeling assumptions merely to obtain a desired result.
-   Suppress validation failures or warnings (e.g. with `suppressWarnings()` or `try()`) solely to allow an analysis to complete.
-   Add `na.rm = TRUE`, `drop_na()` or similar solely to make an error or warning go away.
-   Introduce new dependencies when existing project tools adequately solve the problem.
-   Refactor unrelated analytical code while implementing a requested change.

Automated tools and AI agents **SHOULD** inspect relevant data, code, documentation, and tests before modifying an established analysis.

When multiple reasonable analytical choices exist and the choice could materially affect results, automated tools and AI agents **NEVER** guess the intended approach.

The ambiguity **MUST** be surfaced for an explicit analytical decision.

When unexpected data may represent a legitimate scientific, operational, or data-quality condition, automated tools and AI agents **SHOULD** preserve the data and surface the condition for review rather than automatically correcting it.

The objective of automation is to make analytical work more reliable and reproducible, not to remove necessary scientific judgment.
