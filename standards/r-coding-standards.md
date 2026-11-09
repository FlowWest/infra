# R Coding Standards

## 1. Purpose

These standards define the default conventions for R development across the organization.

The goals are to:

-   Keep projects consistent and maintainable.
-   Reduce unnecessary tooling decisions.
-   Catch errors before code reaches production.
-   Make projects easy for other developers to understand and run.
-   Encourage modern R practices without unnecessary complexity.

Standards specific to data analysis and data science (data provenance, validation, missing data and analytical workflows) are defined in the R Data Analysis & Data Science Standards.

These standards apply to new R projects. Existing projects **MAY** adopt them incrementally.

**Requirement levels.** These terms are capitalized throughout this document and the R Data Analysis & Data Science Standards:

-   **MUST** / **NEVER**: Required / prohibited.
-   **SHOULD** / **AVOID**: Expected unless there is a documented reason not to.
-   **MAY** / **CONSIDER**: Optional.

Plain instructions without one of these terms (e.g. "Use testthat for testing") have **SHOULD** strength.

**Adoption priority for existing projects.** Adopt in this order:

1.  Secrets handling and `.gitignore` (Section 19)
2.  renv for analysis projects (Section 3)
3.  Air and lintr, enforced in CI (Sections 5 and 21)
4.  Tests for new and changed code (Section 8)
5.  All other standards

The default tools are summarized in Section 24.

------------------------------------------------------------------------

## 2. R Version

New projects **SHOULD** use a currently supported R release.

**AVOID** supporting old R versions unless there is a specific business requirement.

------------------------------------------------------------------------

## 3. Project and Dependency Management

Analysis projects **SHOULD** use `renv` to provide:

-   A project-specific package library
-   Dependency management
-   Dependency locking
-   A record of the R version used by the project

Packages, and analyses developed within a package structure (see Section 9), do not need to use renv.

renv records the R version but does not install or switch R versions. Use rig when you need to install or switch between R versions.

Projects that use renv **MUST** commit:

-   `.Rprofile`
-   `renv.lock`
-   `renv/activate.R`
-   `renv/settings.json`
-   `renv/.gitignore`

**NEVER** commit `renv/library/`.

Inside renv projects, install packages with `renv::install()` so the project library and lockfile stay in sync. Outside renv projects, use pak (`pak::pak()`), including for packages hosted on GitHub.

**Packages** declare their dependencies in `DESCRIPTION` using minimum versions (e.g. `dplyr (>= 1.1.0)`) and do not include a `renv.lock`. If renv is used during package development, add `renv.lock` and `renv/` to `.Rbuildignore`.

------------------------------------------------------------------------

## 4. Project Configuration

Use `DESCRIPTION` as the central configuration file when developing packages.

Configuration for the following **SHOULD** live there:

-   Package metadata and dependencies (`Imports`, `Suggests`). **AVOID** `Remotes` in shared packages (see Section 22).

-   testthat settings (`Config/testthat/edition: 3`, `Config/testthat/parallel: true`)

-   roxygen2 settings (`Roxygen: list(markdown = TRUE)`)

-   Development-only dependencies (`Config/Needs/<purpose>`, e.g. `Config/Needs/website`)

Some tools don't support `DESCRIPTION`. Use these dedicated files, kept at the project root:

-   Air (formatting): `air.toml`

-   lintr (linting): `.lintr`

-   Build exclusions: `.Rbuildignore`

-   covr (coverage): `.covrignore`, only if needed

Projects that use renv also commit `renv.lock` (see Section 3). It is managed by renv; **NEVER** edit it by hand.

------------------------------------------------------------------------

## 5. Formatting and Linting

Use `air` for formatting and `lintr` for linting.

Developers **SHOULD** configure their editor to format R files using `air` on save. Supported editors are Positron and RStudio. Follow the Air documentation to enable format-on-save in each.

**AVOID** manually debating formatting conventions that `air` handles automatically.

Air handles layout, but not the following conventions, which **MUST** also be followed:

-   Use the native pipe `|>` for new code. **NEVER** use `%>%` in new code.
-   Use `<-` for assignment, not `=`.
-   Use `TRUE` and `FALSE`, **NEVER** `T` and `F`.

Use the organization's shared `.lintr` template so the same linters are enabled in every project. **AVOID** disabling linters project-wide without a documented reason. Disable individual lines with `# nolint` only when the exception is deliberate, and add a comment explaining why.

------------------------------------------------------------------------

## 6. Type Safety and Input Validation

R does not support static type annotations. Instead, R code **SHOULD** document expected types and validate inputs at run time.

**Documentation.** Use roxygen2 for package development to describe the expected type and shape of each value for:

-   Public (exported) functions

-   Function parameters (`@param`), e.g. `@param flow A numeric vector of daily flows (cfs). Must be non-negative.`

-   Function return values (`@returns`)

-   Important data structures, including the required columns and their types for data frames

**Input validation.** Use **checkmate** as the preferred validation package for new projects, e.g. `checkmate::assert_numeric(flow, lower = 0, any.missing = FALSE)`. Projects **MAY** use base `stopifnot()` or rlang helpers such as `rlang::arg_match()` when:

-   Existing projects already rely on them.

-   The project is trying to keep its dependencies minimal.

-   The check is simple enough that adding a package isn't justified.

**Data structures.** For important objects with a defined structure, S7 classes with typed properties **SHOULD** be preferred over loosely structured lists (see Section 7). For tabular data, validate the required columns and their types when the data is read in, using checkmate or dplyr checks (see the R Data Analysis & Data Science Standards, Section 9). *In development: the use of dedicated validation frameworks such as pointblank is under review.*

Validation is generally unnecessary for internal helper functions whose inputs have already been checked. **AVOID** functions that silently return different types depending on their input, such as `sapply()`. Type-stable alternatives such as `vapply()` or `purrr::map_dbl()` **SHOULD** be used instead.

**CI.** Because R has no static type checker, tests **MUST** confirm that invalid inputs raise informative errors. See Section 21 for the required CI checks.

------------------------------------------------------------------------

## 7. Data Models

Use named lists or tibbles for simple internal data structures. Use tibbles for tabular data and named lists for small groups of related values.

Use S7 classes when data requires:

-   Validation, through typed properties and a `validator` function

-   A stable, documented structure that several functions depend on

-   Methods that behave differently depending on the object type

When data crosses a system boundary, validate it at the point of entry:

-   **Parsing and serialization:** use jsonlite for JSON and yaml for YAML. Validate the parsed result with checkmate before using it.

-   **External APIs:** validate request inputs and responses explicitly, e.g. in plumber endpoints or httr2 response handling. **NEVER** assume the data has the expected structure.

-   **Configuration:** use the config package (`config.yml`) and validate the loaded values with checkmate.

-   **Tabular data from external sources:** check the required columns and their types on read-in, with checkmate or dplyr checks.

**AVOID** S7 classes where a list or tibble is sufficient. Use S7 for new structured classes. Existing S3 or S4 code doesn't need to be migrated.

------------------------------------------------------------------------

## 8. Testing

Use `testthat` (3rd edition) for testing.

Tests **SHOULD** be written for:

-   Business logic
-   Data transformations
-   Bug fixes
-   Important calculations
-   Public APIs
-   Behavior likely to regress

Bug fixes **SHOULD** include a regression test.

**Test file structure.** Tests **MUST** mirror the code one file to one file: `R/estimate-survival.R` is tested in `tests/testthat/test-estimate-survival.R`. Use `usethis::use_r()` and `usethis::use_test()` to create matching pairs.

**Snapshot tests.** Use `expect_snapshot()` to test error messages, warnings and printed output, rather than matching message text with regular expressions.

**Test data.** Keep small test datasets in `tests/testthat/fixtures/` and access them with `testthat::test_path("fixtures", ...)`. Test data **SHOULD** be the smallest dataset that exercises the behavior being tested.

**Property-based testing.** **CONSIDER** `quickcheck` when property-based testing provides meaningful value, especially for:

-   Numerical code
-   Parsers
-   Validation logic
-   Functions with large input spaces

**Determinism.** Tests **SHOULD** be deterministic. Tests that use randomness, including property-based tests, **MUST** set a seed (e.g. `withr::local_seed()`) to keep them deterministic and make failures reproducible.

**AVOID** tests that depend unnecessarily on the following. Use the listed tools when a dependency can't be avoided:

-   Internet access and external services: record and replay HTTP calls with httptest2 or vcr, and use `skip_if_offline()`.
-   Execution order: each test **MUST** set up its own state. Running tests in parallel (`Config/testthat/parallel: true`) will expose order dependencies.
-   Local filesystem state: use `withr::local_tempdir()` for files created during tests.
-   Current time: wrap `Sys.time()` or `Sys.Date()` in your own function and replace it in tests with `testthat::local_mocked_bindings()`.

**Coverage.** Measure test coverage with covr. Coverage is informational: use it to find untested code, not as a target to maximize. [Optional: set a project minimum, e.g. 70%, for shared packages.]

------------------------------------------------------------------------

## 9. Project Structure

There are two project types. Choose the package structure when code is reused across projects or needs to be installed. Choose the analysis structure for reports, model runs and project-specific workflows.

**Packages:**

```         
project/
├── DESCRIPTION
├── NAMESPACE
├── NEWS.md
├── README.Rmd
├── README.md
├── .Rbuildignore
├── .lintr
├── air.toml
├── R/
│   ├── data-import.R
│   ├── estimate-survival.R
│   └── utils.R
├── man/
└── tests/
    ├── testthat.R
    └── testthat/
        ├── fixtures/
        ├── test-data-import.R
        └── test-estimate-survival.R
```

**Analysis projects:**

```         
project/
├── project.Rproj
├── .Rprofile
├── renv.lock
├── renv/
├── README.md
├── .lintr
├── air.toml
├── R/              # project functions
├── analysis/       # scripts and Quarto documents
├── data-raw/       # source data, never modified
├── data/           # processed and derived data
├── outputs/        # figures, tables and deliverables (or results/)
└── tests/
    └── testthat/
```

Analysis projects **MAY** use a folder named `outputs/` or `results/` for figures, tables and deliverables, used consistently within the project. See the R Data Analysis & Data Science Standards, Section 7, for source data and provenance rules.

A complex analysis **MAY** also use the package structure, so that `devtools::load_all()` and testthat work without publishing anything. Analyses developed within a package structure do not need to use renv (see Section 3).

R does not allow subfolders inside `R/`. Group related functions by file name instead, using a shared prefix such as `model-fit.R` and `model-predict.R`.

`NAMESPACE` and the files in `man/` are generated by roxygen2 (`devtools::document()`). **NEVER** edit them by hand.

------------------------------------------------------------------------

## 10. Naming

Follow the tidyverse style guide naming conventions.

-   Variables: `snake_case`
-   Functions: `snake_case`, preferably verbs (`estimate_survival()`, not `survival()`)
-   Files: `kebab-case.R` (e.g. `estimate-survival.R`), matching testthat's `test-*.R` pattern
-   S7 and R6 classes: `PascalCase`
-   S3 classes: `snake_case` (e.g. `class(x) <- "trap_sample"`)
-   Constants: `UPPER_SNAKE_CASE` for fixed values defined once at the top level, such as `MAX_FORK_LENGTH_MM`
-   Internal functions: do not export them (no `@export` in roxygen2). A leading dot (`.helper()`) is optional for marking them.

**NEVER** use dots in function or variable names (e.g. `mean.flow`). R uses dots to define S3 methods, such as `print.trap_sample()`, so a name like `mean.flow` can be mistaken for a method.

**AVOID** reusing the names of common base R functions or values, such as `c`, `t`, `T`, `F`, `mean`, `data`, `df` or `length`.

Names **SHOULD** describe intent rather than implementation details. Prefer `mean_streamflow` over `x` when the additional context improves readability.

Short names are acceptable when their meaning is obvious, such as `for (i in seq_along(sites))`. Use `seq_along()` rather than `1:length(x)`, which produces `c(1, 0)` when `x` is empty.

------------------------------------------------------------------------

## 11. Functions

Functions **SHOULD**:

-   Perform one clear task.
-   Have explicit inputs and outputs.
-   Avoid hidden global state.
-   Remain reasonably small.
-   Return values instead of modifying unrelated state.

Simple functions **SHOULD** be preferred over unnecessarily complex class hierarchies. Use S7 classes only when the guidance in Section 7 applies.

**AVOID** deeply nested control flow where possible.

**CONSIDER** early returns when they improve readability.

More specifically:

-   **NEVER modify global state.** **NEVER** use `<<-`, `assign()` or `setwd()` inside functions. If a function must change `options()`, `par()`, environment variables or the working directory, it **MUST** restore them using `withr::local_*()` or `on.exit()`.

-   **NEVER rely on variables from outside the function.** R silently looks up variables it can't find in the function in the environment where the function was defined, which is usually the global environment. A function that uses a `flow_data` object it was never given still works until that object changes or disappears. Pass every input as an argument.

-   **NEVER load packages inside functions.** Call functions with `pkg::fun()` or declare imports in `DESCRIPTION`. **NEVER** call `library()` or `require()` inside a function.

-   **Return the modified object.** R arguments are copied when modified, so a function can't change its input in place. Write `df <- clean_names(df)`, not `clean_names(df)` expecting `df` to change.

-   **Use `return()` only for early exits.** The last expression is returned automatically, so an explicit `return()` signals an early exit.

-   **Raise informative errors.** Use `cli::cli_abort()` with a message that names the problem argument, rather than letting a function fail deep inside another call.

-   **Separate output from computation.** **AVOID** `print()` and `cat()` in functions that compute values. Use `cli::cli_inform()` for progress output, so users can turn it off with `suppressMessages()`.

-   **Use tidy evaluation correctly.** When a function passes column names to dplyr or ggplot2, use `{{ }}` for arguments and `.data$col` for fixed column names. Otherwise the function may silently use a global variable with the same name.

-   **Prefer vectorized operations to loops when they read clearly.** When a loop is needed, preallocate the result (`out <- vector("list", n)`) instead of growing it with `c()` in each iteration.

------------------------------------------------------------------------

## 12. Documentation

Exported functions, classes and datasets **MUST** be documented with roxygen2 (`#'` comments). Non-trivial internal functions **SHOULD** also be documented, using `@noRd` so they don't appear in the package help pages.

Roxygen documentation **SHOULD** explain information that is not obvious from the function signature. Focus on:

-   Purpose (title and description)

-   Important assumptions (`@details`)

-   Parameters (`@param`), including expected type and units

-   Return behavior (`@returns`), including the type and, for data frames, the columns

-   Errors and warnings the function can raise

-   Units

-   Examples (`@examples`)

Use `@inheritParams` to reuse parameter documentation from another function instead of copying it.

Examples are run by `R CMD check`, so they **MUST** be runnable. Use `@examplesIf` for examples that need internet access or credentials.

Document package datasets in `R/data.R` with `@format`, listing each column, its type and its units. Record the data source with `@source`.

**NEVER** edit files in `man/` or `NAMESPACE` by hand. They are generated by `devtools::document()`.

Use comments to explain *why*, not *what*. **AVOID** comments that simply restate the code.

Prefer:

``` r
# Convert stage from feet to meters before model input.
```

over:

``` r
# Multiply stage by 0.3048.
```

In long scripts, use section headers (`# Load data ----`). They appear in the RStudio and Positron outline and make the script easier to navigate.

**README files** **SHOULD** explain how to:

-   Install the project: `renv::restore()` for analysis projects; `pak::pak("org/repo")` or `devtools::install()` for packages.

-   Run the project, e.g. a `targets::tar_make()` call or the order of scripts.

-   Run tests: `devtools::test()`

-   Run linting and formatting: `lintr::lint_package()` and `air format .`

-   Run the full check: `devtools::check()`

-   Regenerate documentation: `devtools::document()`

For packages, write the README as `README.Rmd` and render it with `devtools::build_readme()`, so the code examples in the README are known to run.

Package changes **SHOULD** be recorded in `NEWS.md`.

------------------------------------------------------------------------

## 13. Logging

Production applications (pipelines, Shiny apps, plumber APIs and scheduled jobs) **SHOULD** use the logger package rather than `print()` or `cat()`.

Use appropriate log levels:

-   `DEBUG`: detailed diagnostic information (`log_debug()`)

-   `INFO`: normal application events (`log_info()`)

-   `WARN`: unexpected but recoverable conditions (`log_warn()`)

-   `ERROR`: operations that failed (`log_error()`)

-   `FATAL`: failures affecting the entire application (`log_fatal()`)

Set the minimum level once at application start (e.g. `log_threshold(INFO)`) rather than inside functions. Send logs to a file or JSON format for scheduled and deployed jobs (`log_appender(appender_file(...))`, `log_layout(layout_json())`).

**AVOID** writing logs directly from package code. Raise conditions with `cli::cli_inform()`, `cli::cli_warn()` and `cli::cli_abort()`, and let the calling application decide what to log.

**NEVER** log sensitive information, such as credentials, tokens, connection strings or personal data. Read credentials from environment variables, and **NEVER** log the output of `Sys.getenv()`.

Interactive scripts and command-line tools **MAY** use cli for intentional user-facing output.

------------------------------------------------------------------------

## 14. Errors and Warnings

**NEVER** silently ignore errors. **AVOID** suppressing warnings; any warning that is suppressed **MUST** be documented (see below).

**AVOID**:

``` r
try(read_trap_data(path), silent = TRUE)

tryCatch(read_trap_data(path), error = function(e) NULL)

suppressWarnings(
  dat <- read_trap_data(path)
)
```

Also **AVOID** `purrr::possibly()` and `purrr::safely()` unless the code afterwards checks for and reports the failures.

Catch specific error classes whenever practical:

``` r
tryCatch(
  read_trap_data(path),
  trapdata_missing_file = function(e) {
    log_warn("Skipping {path}: file not found")
    NULL
  }
)
```

Most base R errors don't have a specific class. **NEVER** catch errors by matching message text, because message wording can change between versions or be translated.

Errors **MUST** either:

-   Be handled appropriately.
-   Be logged and re-raised, adding context with `cli::cli_abort(..., parent = e)` so the original error is preserved.
-   Be allowed to propagate to the appropriate application boundary.

Use `on.exit()` or `withr::defer()` for cleanup (closing database connections, removing temporary files) rather than catching errors only in order to clean up.

Treat warnings as problems to fix, not noise. If a warning is suppressed, it **MUST** be a specific, understood and expected warning, and the code **MUST** include a comment documenting what the warning is and why it is safe to suppress. Tests **SHOULD** fail on unexpected warnings (`expect_no_warning()`).

**CONSIDER** custom error classes (`cli::cli_abort(..., class = "pkgname_error_type")`) when they make error handling significantly clearer. Prefix class names with the package name to avoid collisions with other packages.

------------------------------------------------------------------------

## 15. File and Path Handling

Use fs for file operations (copying, moving, deleting, creating folders, listing contents) because, unlike base R, its functions raise errors when they fail. Base R (`file.path()`, `file.exists()`) is acceptable for building paths and simple existence checks.

Use `fs::path()` or `file.path()` to build file paths. **NEVER** paste strings together to build paths.

Prefer:

``` r
path <- fs::path("data", "results.csv")
```

over:

``` r
path <- paste0("data/", "results.csv")
```

In analysis projects, refer to files relative to the project root using `here::here()`:

``` r
dat <- readr::read_csv(here::here("data-raw", "rbdd_2026.csv"))
```

**NEVER** use `setwd()`, and **NEVER** hard-code absolute paths (such as `"C:/Users/ashley/..."` or `"~/Projects/..."`).

In packages, use `system.file(package = "pkgname")` to access files within the package, and use `testthat::test_path()` inside tests. **NEVER** use `here()` in package code.

**NEVER** assume a specific operating system's path separator. Use forward slashes in any path written as text. R accepts them on Windows, while backslashes must be escaped (`"data\\raw"`) and are a common source of errors.

Use fs functions for other path operations instead of string manipulation, e.g. `fs::path_ext_set()`, `fs::path_file()` and `fs::dir_ls()` rather than `sub()` or `list.files()` with regular expressions.

------------------------------------------------------------------------

## 16. Data and Scientific Computing

Preferred packages include:

| Purpose | Preferred Tool |
|----|----|
| Arrays / numerical computing | Base R (vectors, matrices); Matrix for sparse matrices |
| Tabular data | dplyr and tidyr; data.table where performance is critical |
| Reading files | readr (CSV), readxl (Excel) |
| Local analytical SQL | duckdb with DBI; dbplyr to write dplyr code against DuckDB |
| Columnar data interchange | arrow (Parquet, Feather) (*in development*) |
| Statistical analysis | Base `stats`; mgcv, glmmTMB; Stan via brms or cmdstanr for Bayesian models |
| Scientific computing | Task-specific packages (e.g. deSolve for differential equations); base `optim()` / `integrate()` |
| Machine learning | tidymodels |
| Visualization | ggplot2 |
| Spatial data | sf (vector), terra (raster) |
| Dates and times | lubridate |
| Pipelines / workflow | targets (see the R Data Analysis & Data Science Standards, Section 6) |
| Large distributed workloads | sparklyr |

Additional analysis-specific packages are listed in the R Data Analysis & Data Science Standards, Section 5.

Choose the simplest tool appropriate for the workload.

**NEVER** introduce distributed systems such as Spark when local tools such as DuckDB or data.table can comfortably handle the data. *In development: guidance on arrow and storage solutions for larger-than-memory datasets is under review.*

**Tibbles and data frames.** Use tibbles for tabular data in analysis code and in packages that already depend on the tidyverse. Tibbles always return a tibble when subset, don't partially match column names with `$`, and only recycle values of length 1. Base data frames are acceptable in packages trying to minimize dependencies; when using them, subset with `drop = FALSE` and use `[[` rather than `$` for programmatic column access.

Data units **MUST** be explicit in the following when ambiguity could create errors:

-   Variable and column names (e.g. `flow_cfs`, `fork_length_mm`, `temp_c`)
-   Data schemas
-   Documentation, including `@format` for package datasets
-   Metadata

**CONSIDER** the units package for calculations where unit conversions are error-prone. It attaches units to values, converts them automatically and raises an error when incompatible units are combined.

------------------------------------------------------------------------

## 17. Command-Line Applications

Use targets for multi-step analytical workflows that run repeatedly (see the R Data Analysis & Data Science Standards, Section 6). Use a CLI only for tools that are run with varying arguments or by people who don't use R.

Use optparse as the preferred package for new command-line applications. Rapp **MAY** be used for simple scripts where its lighter syntax is an advantage.

**CONSIDER** cli when formatted terminal output materially improves usability: progress bars, headings, lists and status messages.

**AVOID** introducing multiple CLI frameworks within the same project.

Keep CLI scripts thin. Put the logic in package or project functions and have the script only parse arguments and call those functions, so the logic can be tested with testthat.

Command-line scripts **MUST** exit with a non-zero status on failure, so schedulers and CI can detect errors. Uncaught errors in `Rscript` already do this. **NEVER** catch errors and then exit normally.

------------------------------------------------------------------------

## 18. APIs

For new R HTTP APIs, prefer plumber2 unless project requirements justify another framework. Existing plumber APIs do not need to be migrated.

Use typed parameter annotations for basic type checks and conversion, and validate request contents explicitly with checkmate (see Section 6). For structured request bodies, validate the parsed result before using it, following the same approach as for other external data.

API boundaries **MUST**:

-   Validate incoming data. Reject invalid requests with a 400-level status code and a clear message.

-   Return predictable response structures. Use a consistent JSON shape for both successes and errors. Set the JSON serializer to return scalars as single values, not one-element arrays, which is jsonlite's default.

-   Handle failures explicitly. Set a custom error handler. Return an appropriate status code and log the full error on the server.

-   Avoid exposing internal implementation details. **NEVER** return raw R error messages, stack traces, file paths or SQL to the client.

R processes one request at a time per process. APIs that run slow computations (model fits, large queries) **SHOULD** use asynchronous execution (e.g. mirai or promises) or run multiple worker processes. Deployment platforms such as Posit Connect can manage worker processes for you. If an API needs to handle high request volumes, **CONSIDER** serving precomputed results instead of computing on each request.

Keep endpoint functions thin. Put the logic in package functions so it can be tested with testthat without running the API.

------------------------------------------------------------------------

## 19. Secrets and Configuration

**NEVER** commit:

-   Passwords
-   API keys
-   Access tokens
-   Private credentials
-   Production secrets

Configuration that varies between environments **MUST** be provided through:

-   Environment variables
-   Configuration files excluded from version control
-   Approved secret-management systems

**NEVER** hard-code production credentials.

More specifically:

-   Store project credentials in a project-level `.Renviron` file (`usethis::edit_r_environ(scope = "project")`), which **MUST** be listed in `.gitignore`. Read values with `Sys.getenv("VAR_NAME")`.

-   **NEVER** commit `.Rhistory` or `.RData`. Anything typed in the console, including a pasted password, ends up in `.Rhistory`, and `.RData` can contain connection objects or tokens. Turn off saving the workspace (in RStudio/Positron settings, or `usethis::use_blank_slate()`), and run `usethis::git_vaccinate()` once per machine to add these files to your global `.gitignore`.

-   **NEVER** put secrets in `config.yml`. Use it for settings that aren't sensitive and reference secrets indirectly: `password: !expr Sys.getenv("DB_PASSWORD")`.

-   **CONSIDER** the keyring package for credentials on personal machines, so they're stored in the operating system's credential store instead of a text file.

-   Add OAuth token caches to `.gitignore` (e.g. `.httr-oauth`, `.secrets/`). Packages such as googledrive and httr can write these to the project folder.

-   Check rendered outputs. Quarto and R Markdown documents can print a connection string or token into committed HTML. **NEVER** print or echo credential values in code chunks.

-   For deployed content (Posit Connect, scheduled jobs, CI), set credentials as environment variables in that platform. **NEVER** deploy a `.Renviron` file.

------------------------------------------------------------------------

## 20. Notebooks and Literate Documents

Use Quarto (`.qmd`) for new analysis documents. Existing R Markdown documents do not need to be converted. **AVOID** Jupyter with an R kernel for R projects.

Quarto documents are appropriate for:

-   Exploration
-   Research
-   Visualization
-   Prototyping
-   Communicating analysis and reports

Important reusable logic **SHOULD** be moved into functions in `R/` or a package, and loaded with `devtools::load_all()` or `source()`. Documents **SHOULD** mostly call functions, not define complex logic in chunks.

A document is complete only if it renders successfully from a clean session with `quarto render`. Running chunks interactively in the editor does not count.

**NEVER** make production systems depend on running chunks by hand. Render documents from scheduled jobs or pipelines (`quarto render`, or `tarchetypes::tar_quarto()` within targets).

Use parameterized reports (`params:`) to produce the same report for different sites, years or scenarios. **AVOID** copying and editing a document for each variant.

**AVOID** knitr chunk caching (`cache: true`) for important results, because cached chunks can silently show outdated output. Use targets for expensive computations and have the document read the results.

Documents that use randomness **MUST** set a random seed.

Apply the same path rules as other code: use `here::here()` and **NEVER** `setwd()`. Quarto runs code with the working directory set to the document's folder, so relative paths written for the console will often fail.

Commit the `.qmd` source. Commit rendered outputs only when they are deliverables or published pages, and add generated `*_files/` folders to `.gitignore` otherwise.

------------------------------------------------------------------------

## 21. Continuous Integration

Pull requests **MUST** pass automated checks before merging. Use GitHub Actions workflows from r-lib/actions, set up with `usethis::use_github_action()`.

At minimum, for packages:

1.  Formatting check (`air format --check`)

2.  Linting (`lintr::lint_package()`, configured to fail CI on any lint)

3.  `R CMD check`, including tests, examples and documentation consistency. Treat warnings as failures.

For analysis projects:

1.  Formatting check (`air format --check`)

2.  Linting (`lintr::lint_dir()`)

3.  Restore dependencies with `renv::restore()`

4.  Run tests on project functions (`testthat::test_dir("tests/testthat")`)

5.  Validate the pipeline (`targets::tar_validate()`), or run it on a small test dataset where practical

Example logical pipeline:

```         
format check
  ↓
lint
  ↓
R CMD check (tests, examples, docs)
  ↓
build/deploy (pkgdown site, Posit Connect, etc.)
```

CI **SHOULD** also fail if roxygen documentation is out of date. Run `devtools::document()` and fail if it changes any files.

Developers **SHOULD** be able to run the same checks locally that CI runs remotely:

``` r
air format .                # in a terminal
lintr::lint_package()
devtools::check()
```

Projects **MAY** use the precommit R package to run formatting and linting automatically before each commit.

------------------------------------------------------------------------

## 22. Dependencies

Add dependencies intentionally.

Before adding a package, **CONSIDER**:

-   Is the functionality already available in base R or the recommended packages that ship with R (stats, utils, tools, Matrix, MASS)?

-   Is the package actively maintained? Check its last release, open issues, and whether it's passing CRAN's checks. Archived CRAN packages and abandoned GitHub repositories are a risk.

-   Does the dependency materially simplify the code?

-   Is another dependency already providing the same functionality?

-   How many other packages does it pull in? Check with `pak::pkg_deps_tree("pkg")`.

-   Does it require system libraries (e.g. sf needs GDAL, GEOS and PROJ; rJava needs Java)? These make installation and deployment harder, especially on servers and in CI.

In packages:

-   Add dependencies with `usethis::use_package()`, which places them correctly in `DESCRIPTION`.

-   Put packages needed only for tests, vignettes, examples or optional features in `Suggests`, not `Imports`. Check that they're installed before using them (`rlang::check_installed()`).

-   **NEVER** list `tidyverse` as a dependency. Depend on the specific packages you use (dplyr, tidyr, etc.).

-   **AVOID** packages available only on GitHub (`Remotes:`) in shared packages. Prefer CRAN or R-universe releases.

-   Call functions with `pkg::fun()`, or use `@importFrom pkg fun` for functions used frequently. **NEVER** import a whole namespace with `@import`.

Remove unused dependencies. `R CMD check` reports imported packages that are never used. In analysis projects, use `renv::dependencies()` to see what is actually used and `renv::clean()` to remove unused packages from the project library.

**AVOID** introducing multiple packages that solve the same problem without a clear reason. Common examples in R are mixing dplyr and data.table in the same code, or using stringr and base regex functions interchangeably.

------------------------------------------------------------------------

## 23. Code Review

Pull requests **SHOULD** prioritize:

1.  Correctness, including scientific and analytical correctness
2.  Readability
3.  Maintainability
4.  Tests
5.  Input validation and type stability
6.  Reproducibility
7.  Performance where relevant

**AVOID** spending significant review time debating formatting issues handled automatically by tooling.

Small, focused pull requests **SHOULD** be preferred over large unrelated changes. Refactoring **SHOULD** be kept separate from changes that affect results.

**Pull request descriptions** **SHOULD** explain why the change was made, not just what changed. If the change affects results, describe how outputs changed and why. Include before and after values or figures where useful.

**Analytical code review.** For changes to data processing or models, reviewers **SHOULD** check:

-   **Joins:** Is the expected number of rows preserved? Are there unintended duplicates? Joins **SHOULD** declare their expected relationship (see the R Data Analysis & Data Science Standards, Section 9). If they don't, **NEVER** ignore join warnings (e.g. many-to-many); how they were handled **MUST** be documented.

-   **Missing values:** Are `NA`s handled deliberately? Watch for `na.rm = TRUE` silently dropping data, and filters that remove `NA` rows without anyone noticing.

-   **Units and conversions:** Are units consistent and labeled?

-   **Dates and times:** Are time zones correct, and are water-year or season boundaries handled as intended?

-   **Factors and categories:** Are levels complete and correctly ordered? Are unexpected categories handled?

-   **Assumptions:** Are model or method assumptions documented and appropriate for the data?

-   **Randomness:** Are seeds set so results can be reproduced?

Model and method changes **SHOULD** be reviewed by someone with relevant domain expertise, in addition to a code reviewer.

Reviewers **SHOULD** look at rendered outputs (reports, figures, tables) for changes that affect them, not only the code diff.

------------------------------------------------------------------------

## 24. Default Toolchain

The default R development stack is:

| Purpose | Tool |
|----|----|
| R version management | rig (when multiple R versions are needed) |
| Dependency management | renv (analysis projects); `DESCRIPTION` (packages) |
| Package installation | pak; `renv::install()` inside renv projects |
| Project configuration | `DESCRIPTION` |
| Formatting | Air |
| Linting | lintr |
| Type checking | None; runtime input validation instead |
| Input validation | checkmate |
| Tabular data validation | checkmate / dplyr checks (dedicated frameworks such as pointblank *in development*) |
| Testing | testthat (3rd edition) |
| Property testing | quickcheck |
| Structured data models | S7 |
| Simple data structures | named lists, tibbles |
| Documentation | roxygen2 |
| Messages and errors | cli |
| Logging | logger |
| File paths | here (project root), fs (file operations) |
| Notebooks and reports | Quarto |
| Pipelines | targets |
| CLI development | optparse |
| CLI presentation | cli |
| HTTP APIs | plumber2 |
| Continuous integration | GitHub Actions (r-lib/actions) |

Projects **SHOULD** deviate from these defaults only when there is a clear technical reason.

------------------------------------------------------------------------

## 25. AI-Assisted Development

These standards apply equally to code written by people, code written with AI assistance, and code written by AI coding agents. For analytical work, the AI safeguards in the R Data Analysis & Data Science Standards, Section 11, also apply.

**Responsibility.** The person who submits a pull request is responsible for all code in it, however it was produced. AI-generated code **MUST** be read, understood and tested by that person before it is submitted, and **MUST** go through the same review and CI checks as any other code.

**Data and secrets.** **NEVER** provide secrets, credentials, or confidential client or personal data to AI tools that are not approved for that data. Use only AI tools approved by the organization, and follow any client contract terms that restrict how project data is shared.

**Verification.** AI tools can suggest packages, functions or arguments that don't exist or behave differently than described. Before relying on suggested code:

-   Confirm that suggested packages exist on CRAN, R-universe or Bioconductor, are actively maintained, and meet Section 22. Packages with similar names to well-known packages **MUST** be checked carefully.
-   Confirm that functions and arguments exist and behave as described, using the package documentation.
-   Confirm that statistical or analytical methods are appropriate. AI-generated code that runs without errors is not evidence that it is correct.

**AI coding agents** working in a repository **MUST**:

-   Follow existing project conventions and the toolchain in Section 24.
-   Run formatting, linting and tests before proposing changes, and report any failures they could not fix.

AI coding agents **NEVER**:

-   Introduce new dependencies without explicit approval.
-   Refactor or reformat unrelated code while implementing a requested change.
-   Disable linters, skip or delete tests, or weaken test assertions to make checks pass.
-   Suppress errors or warnings (e.g. with `try()`, `suppressWarnings()` or `# nolint`) to make code run or checks pass.
-   Commit secrets, credentials, `.Renviron`, `.Rhistory` or `.RData`.
-   Modify source data, `renv.lock`, `NAMESPACE` or files in `man/` by hand.

When a requirement is ambiguous and different reasonable choices would materially change behavior, agents **MUST** surface the ambiguity for a decision rather than guess.

**Project instructions.** Repositories **SHOULD** include an instruction file for AI agents (e.g. `AGENTS.md`, `CLAUDE.md` or `.github/copilot-instructions.md`) that points to these standards and lists the project's commands for formatting, linting, testing and running the analysis.

**Disclosure.** **CONSIDER** noting in the pull request description when substantial portions of a change were generated by AI, so reviewers can adjust their review.

------------------------------------------------------------------------

## Guiding Principle

Prefer boring, readable, validated, tested R over clever R.

Code should be easy for another developer to understand, run, test, and modify.
