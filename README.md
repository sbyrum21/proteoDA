
<!-- README.md is generated from README.Rmd. Please edit README.Rmd file -->

# proteoDA

<!-- badges: start -->

[![R-CMD-check](https://github.com/ByrumLab/proteoDA/actions/workflows/R-CMD-check.yaml/badge.svg)](https://github.com/ByrumLab/proteoDA/actions/workflows/R-CMD-check.yaml)
[![Codecov test
coverage](https://codecov.io/github/ByrumLab/proteoDA/branch/main/graph/badge.svg)](https://app.codecov.io/github/ByrumLab/proteoDA?branch=main)
<!-- badges: end -->

proteoDA is a streamlined, user-friendly R package for the analysis of
high resolution mass spectrometry protein data. The package uses a
custom S3 class that keeps the R objects consistent across the pipeline
and is easily pipe-able, so minimal R knowledge is required.

## Installation

`proteoDA` is not yet on CRAN, but it is available for install from
GitHub via the `devtools` package. Install `devtools` if you haven’t
already:

``` r
install.packages("devtools")
```

Then install `proteoDA`:

``` r
devtools::install_github("ByrumLab/proteoDA", 
                         dependencies = TRUE, 
                         build_vignettes = TRUE)
```

Using the `build_vignettes = TRUE` argument will build the tutorial
vignette when you install, which you can access by running
`browseVignettes(package = "proteoDA")`. However, building the vignettes
requires some additional software dependencies. If you run into issues
when installing the vignettes, you can set `build_vignettes = FALSE` and
find a pre-built `.html` version of the tutorial in the `vignettes`
folder on [GitHub](https://github.com/ByrumLab/proteoDA).

Once `proteoDA` is installed, load it into R:

``` r
library(proteoDA)
```

## Workflow

![proteoDA workflow
flowchart](./data-raw/proteoDA_flowchart.png?raw=true)

## Example pipeline

Here’s an example pipeline, going from data import to final results. For
a detailed explanation of the pipeline, check out the tutorial vignette:

``` r
# Load data
input_data <- read.csv(system.file("extdata/DIA_data.csv.gz", package = "proteoDA"))
sample_metadata <- read.csv(system.file("extdata/metafile.csv", package = "proteoDA"))

# Split input data into protein intensity data and annotation data
intensity_data <- input_data[,5:21] # select columns 5 to 21
annotation_data <- input_data[,1:4] # select columns 1 to 4
# Match up row names of metadata with column names of data
rownames(sample_metadata) <- sample_metadata$data_column_name

# Assemble into DAList
raw <- DAList(data = intensity_data,
              annotation = annotation_data,
              metadata = sample_metadata)

# Filter out unneeded samples and proteins with too much missing data
filtered <- raw |>
  filter_samples(group != "Pool") |>
  zero_to_missing() |>
  filter_proteins_by_proportion(min_prop = 0.66,
                                grouping_column = "group")
# Make the normalization report
write_norm_report(filtered,
                  grouping_column = "group")

# Normalize
normalized <- normalize_data(filtered, 
                             norm_method = "cycloess")

# Make the quality control report
write_qc_report(normalized,
                color_column = "group")

# Turn metadata column into a factor with desired levels
normalized$metadata$group <- factor(normalized$metadata$group, 
                                    levels = c("normal", "cancer"))

# Add a statistical design, fit the model, and extract results
final <- normalized |>
  add_design(design_formula = ~ group) |>
  fit_limma_model() |>
  extract_DA_results()

# Export results
write_limma_tables(final)
write_limma_plots(final,
                  grouping_column = "group")
```
