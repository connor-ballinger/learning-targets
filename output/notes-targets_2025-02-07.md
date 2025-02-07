---
title: "Notes for `targets`"
date: "7th February, 2025"
author: "Connor Ballinger"
knit: conr::write_and_date
output: 
  conr::format_html:
    keep_md: true
---


``` r
knitr::opts_knit$set(root.dir = here::here())
knitr::opts_chunk$set(message = FALSE, warning = FALSE, comment = "")

library(conr)
library(knitr)
library(tidyverse)
library(targets)

conr::knit_df()
```

[The {targets} R package user manual](https://books.ropensci.org/targets/)

- Start with `use_targets()` to generate the target script. Note `use_targets_rmd` for later use.

All target script files have these requirements:

1. Load the packages needed to define the pipeline, e.g. `targets` itself.

1. Use `tar_option_set()` to declare the packages that the targets themselves need, as well as other settings such as the default storage format.

1. Load your custom functions and small input objects into the R session: in our case, with `source("R/functions.R")`.

1. Write the pipeline at the bottom of `_targets.R`. 

  i. A pipeline is a list of target objects, which you can create with `tar_target()`. 
  i. Each target is a step of the analysis. It looks and feels like a variable in R, but during `tar_make()`, it will reproducibly store a value in `_targets/objects/`.


- `tar_manifest()` lists verbose information about each target.

- `tar_visnetwork()` displays the dependency graph of the pipeline.

- `tar_make()` runs the pipeline.

- The output of the pipeline is saved to the` _targets/` data store, and you can read the output with `tar_read()` (see also `tar_load()`).

(Info on changes...)

- If we change the data file `data.csv`, `targets` notices the change. This is because file is a file target (i.e. with `format = "file"` in `tar_target()`), and the return value from last `tar_make()` identified `"data.csv"` as the file to be tracked for changes. 

# Debugging

- R code is easiest to debug when it is interactive. 

- However, a pipeline is the opposite of interactive. In targets, several layers of encapsulation and automation separate you from the code you want to debug.

- Even if you hit an error, you can still finish the successful parts of the pipeline. For example, tar_option_set(error = "null") tells errored targets to return NULL.  The output as a whole will not be correct or up to date, but the pipeline will finish so you can look at preliminary results.

## Error Messages

- _targets/meta/meta stores the most recent error and warning messages for each target. Access it with tar_meta().

## Debugging Functions

- Most errors come from the user-defined functions.

- dunno, come back to it.

# Functions

# Targets

- A target is a high-level step of the computational pipeline, and a piece of work that you define with your custom functions. 

- A target runs some R code and saves the returned R object to storage, usually a single file inside _targets/objects/.

- A target is an abstraction.

- you do not need to reference a target’s data file directly.  Instead, your R code should refer to a target name as if it were a variable in an R session. In other words, from the point of view of the user, a target is an R object in memory.

- Like a good function, a good target generally does one of three things:

  1. Create a dataset.
  1. Analyze a dataset with a model.
  1. Summarize an analysis or dataset.

- It is best to define targets that maximize time savings. Good targets usually:

  1. Are large enough to subtract a decent amount of runtime when skipped.
  1. Are small enough that some targets can be skipped even if others need to run.
  1. Invoke no side effects such as modifications to the global environment. (But targets with tar_target(format = "file") can save files.)
  1. Return a single value that is meaningful.

- Like a good function, a good target should return a single value and not produce side effects. (The exception is output file targets which create files and return their paths.)

# `renv`

- `renv` is recommended but can cause speed issues which may be managed by tinkering with the `renv` config. Sections 7.1 and 8.1 address this.

# Personal Packages

- if you are developing a package alongside a targets pipeline that uses it, you may wish to invalidate certain targets as you make changes to your package.

(Not sure what this section means... Basically a way of tracking changes in your personal package, but wouldn't this be the same as an update to any other package - the package version will change, so the pipeline needs updating? Maybe not, maybe this is for non-renv projects and targets doesn't track when they get updated? Overally, reassuring that this is something which is addressed, don't think it will be an issue.)

# Projects

- The entire _targets/ data store should generally not be committed to Git because of its large size.

- Example structure:

├── .git/
├── .Rprofile
├── .Renviron
├── renv/
├── index.Rmd
├── _targets/
├── _targets.R
├── _targets.yaml
├── R/
├──── functions_data.R
├──── functions_analysis.R
├──── functions_visualization.R
├── data/
└──── input_data.csv

- index.Rmd: Target Markdown report source file to define the pipeline.

- _targets/: the data store where tar_make() and similar functions write target storage and metadata when they run the pipeline.

- _targets.R: the target script file. All targets pipelines must have a target script file that returns a target list at the end. If you use Target Markdown (e.g. index.Rmd above) then the target script will be written automatically. Otherwise, you may write it by hand.

- _targets.yaml: a YAML file to set default arguments to critical functions like tar_make(). As described below, you can access and modify this file with functions tar_config_get(), tar_config_set(), and tar_config_unset(). 

# Local Data

- targets makes use of RAM.

- After a target runs or loads, tar_make() either keeps the object in memory or discards it, depending on the settings in tar_target() and tar_option_set().

- The pipeline also writes to disk.

- Note:

  - tar_read(target1) is much better than readRDS("_targets/objects/target1")

    - A file target can have both input and output files.

     - There are multiple functions to remove or clean up local target storage.

# Literate programming

- There are two kinds of literate programming in targets:

1. A literate programming source document (or Quarto project) that renders inside an individual target. Here, you define a special kind of target that runs a lightweight R Markdown report which depends on upstream targets.

1. Target Markdown, an overarching system in which one or more Quarto or R Markdown files write the _targets.R file and encapsulate the pipeline.

- We recommend (1) in order to fully embrace pipelines as a paradigm.

## Rmarkdown targets

- Here, literate programming serves to display, summarize, and annotate results from upstream in the targets pipeline. 

- The document(s) have little to no computation of their own, and they make heavy use of tar_read() and tar_load() to leverage output from other targets.

- The .Rmd can be included as a target.

- The report can be re-rendered in the pipeline whenever fit or hist changes, which means tar_make() brings the output file report.html up to date.

- tar_render() is like tar_target(), except that you supply the file path to the R Markdown report instead of an R command. 

- example:


``` r
## _targets.R ------------------------------------------------------------------
# do not run

library(targets)
library(tarchetypes)
source("R/functions.R")
list(
  tar_target(
    raw_data_file,
    "data/raw_data.csv",
    format = "file"
  ),
  tar_target(
    raw_data,
    read_csv(raw_data_file, col_types = cols())
  ),
  tar_target(
    data,
    raw_data %>%
      mutate(Ozone = replace_na(Ozone, mean(Ozone, na.rm = TRUE)))
  ),
  tar_target(hist, create_plot(data)),
  tar_target(fit, biglm(Ozone ~ Wind + Temp, data)),
  tar_render(report, "report.Rmd") # Here is our call to tar_render().
)
```

## Quarto targets

- tar_quarto() works exactly the same way as tar_render(). However, tar_quarto() is more powerful: you can supply the path to an entire Quarto project, such as a book, blog, or website. 

- tar_quarto() looks for target dependencies in all the source documents (e.g. listed in _quarto.yml), and it tracks the important files in the project for changes (run tar_quarto_files() to see which ones).

- Note documents can be parameterised.

# END
