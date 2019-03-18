
# Howto

## Generating vignettes output as markdown

```R
# cd ~/src/csiro/stash/xxxxxx/swift/bindings/R/pkgs/swift/vignettes
# R

setwd('C:/src/csiro/stash/swift/bindings/R/pkgs/swift/vignettes')

library(rmarkdown)

infn <- c(
'calibrate_subcatchments'
,'calibration_initial_states' 
,'ensemble_model_runs'
,'error_correction_four_stages'
,'getting_started'
,'log_likelihood'
,'meta_parameters'
,'muskingum_multilink_calibration'
)


f <- function(fn)
{
    input <- paste(fn, 'Rmd', sep='.')
    output_format <- 'github_document'

    output_dir <- fn

    rmarkdown::render(input, output_format, output_file = NULL, output_dir,
                output_options = NULL, intermediates_dir = NULL,
                knit_root_dir = NULL,
                runtime = c("auto", "static", "shiny", "shiny_prerendered"),
                clean = TRUE, params = NULL, knit_meta = NULL,
                run_pandoc = TRUE, quiet = FALSE)
}

lapply(infn, FUN=f)


file.remove(list.files('.', pattern="*.html", full.names=TRUE, recursive=TRUE))
file.copy(infn, '~/src/github_jm/streamflow-forecasting-tools-onboard/doc/vignettes/', recursive=TRUE)

file.copy(infn, 'c:/src/github_jm/streamflow-forecasting-tools-onboard/doc/vignettes/', recursive=TRUE)
```

given what I get from [this issue](https://github.com/rstudio/rmarkdown/issues/587) I am not sure it is possible to get relative paths to figures in the markdown documents. Have to use full text search/replace to correct. 

Note the regex pattern to use:

```text
/home/xxxxxx/src/csiro/stash/xxxxxx/swift/bindings/R/pkgs/swift/vignettes/[a-z_]*/
```
