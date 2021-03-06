Graphic Summaries of of General Water Quality (Sonde) Data From FOCB
================
Curtis C. Bohlen, Casco Bay Estuary Partnership
2/18/2021

-   [Introduction](#introduction)
-   [Load Libraries](#load-libraries)
-   [Load Data](#load-data)
    -   [Establish Folder Reference](#establish-folder-reference)
    -   [Load The Data](#load-the-data)
        -   [Primary Data](#primary-data)
    -   [Transformed Chlorophyll Data](#transformed-chlorophyll-data)
    -   [Create Long Form Data](#create-long-form-data)
    -   [Create Daily Data Summaries](#create-daily-data-summaries)
-   [Three Plotting Function](#three-plotting-function)
    -   [Plot Versus Time](#plot-versus-time)
    -   [Season Profile or Climatology](#season-profile-or-climatology)
    -   [Cross Plots](#cross-plots)
        -   [Helper Function to Add Medians by Color
            Groups](#helper-function-to-add-medians-by-color-groups)
-   [Values by Time](#values-by-time)
    -   [Faceted Plot](#faceted-plot)
        -   [Guide Functions to Transform Chlorophyll
            Axis](#guide-functions-to-transform-chlorophyll-axis)
-   [Seasonal Profiles](#seasonal-profiles)
    -   [Example: Chlorophyll A](#example-chlorophyll-a)
    -   [Joint Plot](#joint-plot)
    -   [Revised Plot](#revised-plot)
-   [Cross-Plots](#cross-plots-1)
    -   [Dissolved Oxygen and
        Temperature](#dissolved-oxygen-and-temperature)
        -   [Add Mpnthly Medians](#add-mpnthly-medians)
        -   [Compare Daily Data](#compare-daily-data)

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />

# Introduction

This Notebook provides graphic summaries of data from Friends of Casco
Bay’s “CMS1” monitoring location, on Cousins Island, in Casco Bay,
Maine. We focus here on analysis of primary sonde data on temperature,
dissolved oxygen, salinity, and chlorophyll A.

Most graphics developed here were not included in the most recent State
of Casco Bay Report, but we retain the code because these graphical
summaries are especially useful for exploratory data analysis of data
from loggers.

# Load Libraries

``` r
library(tidyverse)
#> Warning: package 'tidyverse' was built under R version 4.0.5
#> -- Attaching packages --------------------------------------- tidyverse 1.3.1 --
#> v ggplot2 3.3.5     v purrr   0.3.4
#> v tibble  3.1.6     v dplyr   1.0.7
#> v tidyr   1.1.4     v stringr 1.4.0
#> v readr   2.1.0     v forcats 0.5.1
#> Warning: package 'ggplot2' was built under R version 4.0.5
#> Warning: package 'tidyr' was built under R version 4.0.5
#> Warning: package 'dplyr' was built under R version 4.0.5
#> Warning: package 'forcats' was built under R version 4.0.5
#> -- Conflicts ------------------------------------------ tidyverse_conflicts() --
#> x dplyr::filter() masks stats::filter()
#> x dplyr::lag()    masks stats::lag()
library(readxl)

library(GGally)
#> Warning: package 'GGally' was built under R version 4.0.5
#> Registered S3 method overwritten by 'GGally':
#>   method from   
#>   +.gg   ggplot2
library(lubridate)  # here, for the make_datetime() function
#> Warning: package 'lubridate' was built under R version 4.0.5
#> 
#> Attaching package: 'lubridate'
#> The following objects are masked from 'package:base':
#> 
#>     date, intersect, setdiff, union

library(colorspace)  #for scale_color_continuous_diverging(palette = 'Cork'...) possibly others, 
#> Warning: package 'colorspace' was built under R version 4.0.5

library(CBEPgraphics)
load_cbep_fonts()
theme_set(theme_cbep())
```

# Load Data

## Establish Folder Reference

``` r
sibfldnm <- 'Data'
parent   <- dirname(getwd())
sibling  <- file.path(parent,sibfldnm)

dir.create(file.path(getwd(), 'figures'), showWarnings = FALSE)
```

## Load The Data

We need to skip the second row here, which is inconvenient largely
because the default “guess” of data contents for each column is based on
the contents of that first row of data.

A solution in an answer to this stack overflow questions
<https://stackoverflow.com/questions/51673418/how-to-skip-the-second-row-using-readxl>)
suggests reading in the first row only to generate names, then skip the
row of names and the row of units, and read the “REAL” data. Note that
we round the timestamp on the data to the nearest hour.

R does not read the dates and times correctly, because dates were
entered using inconsistent conventions. Some dates and times are
formatted Excel Dates, some are character strings. WE found it simplest
to reconstruct dates and times from time components, which were provided
as well.

### Primary Data

``` r
fn    <- 'CMS1 Data through 2019.xlsx'
fpath <- file.path(sibling,fn)

mynames <- read_excel(fpath, sheet = 'Sheet1', n_max = 1, col_names = FALSE)
#> New names:
#> * `` -> ...1
#> * `` -> ...2
#> * `` -> ...3
#> * `` -> ...4
#> * `` -> ...5
#> * ...
mynames <- unname(unlist(mynames[1,]))  # flatten and simplify
mynames[2] <- 'datetime'               # 
mynames[4] <- 'depth'                   # Address non-standard names
mynames[8] <- 'pctsat'
mynames[18] <- 'omega_a'
mynames <- tolower(mynames)             # convert all to lower case

the_data <- read_excel(fpath, skip=2, col_names = FALSE)
#> New names:
#> * `` -> ...1
#> * `` -> ...2
#> * `` -> ...3
#> * `` -> ...4
#> * `` -> ...5
#> * ...
names(the_data) <- mynames
rm(mynames)
```

We create an independent time stamp based on recorded year, month, day,
etc. to directly address ambiguities of how dates and times are imported
with daylight savings time, etc. The parameter `tz = "America/New_York"`
creates a time stamp that is tied to local time. The time stamp under
the hood is a numerical value, but with this timezone specification, the
text form accounts for local daylight savings time.

``` r
the_data <- the_data %>%
  select(-count)  %>%       # datetime and time contain the same data
  mutate(dt = make_datetime(year, month, day, hour, 0, tz = "America/New_York")) %>%
  relocate(c(ta, dic, omega_a) , .after = "pco2") %>%
  mutate(thedate  = as.Date(dt),
         doy      = as.numeric(format(dt, format = '%j')),
         # tstamp   = paste0(year, '/', sprintf("%02d", month), '/',
         #                   sprintf("%02d", day), ' ', sprintf("%02d", hour)),
         Month = factor(month, labels = month.abb)) %>%
  arrange(dt)                # Force data are in chronological order
```

## Transformed Chlorophyll Data

For our data based on FOCB’s surface water (grab sample) data, we
presented analyses not of raw chlorophyll data, but analysis of log
(Chlorophyll + 1) data. The transformed values better correspond to
assumptions of normality used is statistical analyses. We provide a
transformed version here so that we can produce graphics that are
visually consistent in terms of presentation.

``` r
the_data <- the_data %>%
  mutate(chl_log1 = log1p(chl)) %>%
  relocate(chl_log1, .after = chl)
```

## Create Long Form Data

``` r
long_data <- the_data %>%
  pivot_longer(cols= depth:omega_a, names_to='Parameter', values_to = 'Value') %>%
  mutate(Parameter = factor(Parameter,
                            levels = c('depth',
                                       'temperature',
                                       'salinity',
                                       'do',
                                       'pctsat',
                                       'chl',
                                       'chl_log1',
                                       'ph',
                                       'pco2',
                                       'ta',
                                       'dic',
                                       'omega_a')))
```

## Create Daily Data Summaries

``` r
daily_data <- the_data %>%
  select(-hour, -year, -month, -day, -doy) %>%         # Will recalculate these 
  group_by(thedate) %>%
  summarise_at(c("temperature", "salinity", "do", "pctsat", "chl", "chl_log1", 
                 "ph", "pco2", "ta", "dic", 'omega_a'),
               c(avg    = function(x) mean(x, na.rm=TRUE),
                 med    = function(x) median(x, na.rm=TRUE),
                 rng    = function(x) {suppressWarnings(max(x, na.rm=TRUE) -
                                                        min(x, na.rm=TRUE))},
                iqr  = function(x) IQR(x, na.rm=TRUE),
                p80r = function(x) {as.numeric(quantile(x, 0.90, na.rm=TRUE) -
                                               quantile(x, 0.10, na.rm=TRUE))}
                )) %>%
  # We recalculate time metrics (has to be outside of `summarise_at()`)
  mutate(year = as.numeric(format(thedate, format = '%Y')),
         month  = as.numeric(format(thedate, format = '%m')),
         day   = as.numeric(format(thedate, format = '%d')),
         doy  = as.numeric(format(thedate, format = '%j')),
         Month = factor(month, levels=1:12, labels = month.abb)
         )
```

# Three Plotting Function

[Back to top](#) We construct three functions. Each produces a useful
graphic for examining long-term data from data loggers, with a
consistent graphic style.

1.  The first generates a plot showing value against time over the whole
    period of record. This is most useful for data QA/QX or for
    depicting long-term trends.

2.  The second shows a “Seasonal Profile” or Climatology. This is best
    for showing data from just a few years at a time.

3.  The third plots one measured variable against another, with a third
    possibly coded via color. This is best for exploratory data
    analysis, or to emphasize a specific pair-wise pattern.

These functions make partial use of the tidyverse’s quosures, but do not
take full advantage of ther capabilities. They should be revised to use
`eval_tidy()` from “rlang”.

## Plot Versus Time

``` r
full_profile <- function(dat, parm, dt = 'dt', color = 'temperature', 
                           label = NA, color_label = '',
                           guide = NA, h_adjust = 0.25,
                           alpha = 0.25, size = 0.5,
                           add_smooth = FALSE, with_se = FALSE) {

  # These are ugly argument checks, since they don't provide nice error messages.
  stopifnot(is.data.frame(dat))
  stopifnot(is.na(label) || (is.character(label) && length(label) == 1))
  stopifnot(is.na(guide) || (is.numeric(guide)   && length(guide) == 1))
  
  # Convert passed parameters to strings, if they are not passed as strings.
  parm <- enquo(parm)
  dt <- enquo(dt)
  color <- enquo(color)
  
  parmname <- rlang::as_name(parm)
  dtname  <- rlang::as_name(dt)
  colorname <- rlang::as_name(color)
  
  # Check that these are data names from the data frame
  stopifnot(parmname %in% names(dat))
  stopifnot(dtname %in% names(dat))  
  stopifnot(colorname %in% names(dat))  

  # Create the variables we will actually pas to ggplot
  x   <- dat[[dtname]]
  y   <- dat[[parmname]]
  col <- dat[[colorname]]
  
  
  plt <- ggplot(dat, aes(x, y)) +
    geom_point(aes(color = col), alpha = alpha, size = size) +
    xlab('') +
    ylab(parmname) +

    guides(colour = guide_legend(override.aes = list(alpha = 1, size = 2),
                                 byrow = TRUE)) +
    
    theme_cbep(base_size = 12) +
    theme(legend.position =  'bottom',
          legend.title = element_text(size = 10))
  
  if(add_smooth) {
         plt <- plt + geom_smooth(method = 'gam', 
                                  formula = y ~ s(x, bs = "cc"),
                                  se = with_se,
                                  color = cbep_colors()[1])
  }
  
  if (is.factor(col)) {
    plt <- plt +
        scale_color_viridis_d(name = color_label, option = 'viridis')

  }
  else {
    plt <- plt +
        scale_color_viridis_c(name = color_label, option = 'viridis')
  }
  
  if(! is.na(guide)) {
         #lab = if_else(is.na(label), parmname, gsub('\\s*\\([^\\)]+\\)','', label))
         plt <- plt + geom_hline(aes(yintercept = guide), 
                                 lty = 'dotted', color = 'gray15') # 
                      # annotate('text', x=min(x), y=(1 + h_adjust) *(guide), 
                      #          label=  round(guide,2),
                      #                       hjust = 0, size=3 )
         }
  
  if(! is.na(label)) {
         plt <- plt + ylab(label)
         }

  return(plt)
}
```

## Season Profile or Climatology

This uses our standard `CBEP_colors()`, but reordered. This corresponds
to ordering in the OA chapter.

``` r
season_profile <- function(dat, parm, doy = 'doy', year = "year", 
                           label = NA, guide = NA, h_adjust = 0.05,
                           alpha = 0.25, size = 0.5,
                           add_smooth = FALSE, with_se = FALSE) {

  # These are ugly argument checks, since they don't provide nice error messages.
  stopifnot(is.data.frame(dat))
  stopifnot(is.na(label) || (is.character(label) && length(label) == 1))
  stopifnot(is.na(guide) || (is.numeric(guide)   && length(guide) == 1))
  
  # Flexible labeling for months of the year
  monthlengths <-  c(31,28,31, 30,31,30,31,31,30,31,30,31)
  cutpoints    <- c(0, cumsum(monthlengths)[1:12])
  monthlabs    <- c(month.abb,'')
  
  # Convert passed parameters to strings, if they are not passed as strings.
  parm <- enquo(parm)
  doy <- enquo(doy)
  year <- enquo(year)
  
  parmname <- rlang::as_name(parm)
  doyname  <- rlang::as_name(doy)
  yearname <- rlang::as_name(year)
  stopifnot(parmname %in% names(dat))
  stopifnot(doyname %in% names(dat))  
  stopifnot(yearname %in% names(dat))  

  x   <- dat[[doyname]]
  y   <- dat[[parmname]]
  col <- dat[[yearname]]
  
  plt <- ggplot(dat, aes(x, y)) +
    geom_point(aes(color = factor(col)), alpha = alpha, size = size) +
    xlab('') +
    ylab(parmname) +
    
    scale_color_manual(values=cbep_colors()[c(3,2,5,4,6)], name='Year') +
    scale_x_continuous(breaks = cutpoints, labels = monthlabs) +
    guides(colour = guide_legend(override.aes = list(alpha = 1, size = 2))) +
    
    theme_cbep(base_size = 12) +
    theme(axis.text.x=element_text(angle=90, vjust = 1.5)) +
    theme(legend.position =  'bottom',
          legend.title = element_text(size = 10))
  
  
  
  if(add_smooth) {
         plt <- plt + geom_smooth(method = 'gam', 
                                  formula = y ~ s(x, bs = "cc"),
                                  se = with_se,
                                  color = cbep_colors()[1])
         }
  
  if(! is.na(guide)) {
         lab = if_else(is.na(label), parmname, gsub('\\s*\\([^\\)]+\\)','', label))
         plt <- plt + geom_hline(aes(yintercept = guide), 
                                 lty = 'dotted', color = 'gray15') +
                      annotate('text', x=0, y=(1 + h_adjust) *(guide), 
                               label= paste(lab, '=', guide),
                                            hjust = 0, size=3)
         }
  
  if(! is.na(label)) {
         plt <- plt + ylab(label)
         }

  return(plt)
}
```

## Cross Plots

``` r
cross_plot <- function(dat, x_parm, y_parm, color_parm = "Month",
                            x_label = NA, y_label = NA, color_label = '',
                            alpha = 0.25,
                            size = 0.5,
                            add_smooth = FALSE,
                            with_se = FALSE) {

  # These are ugly argument checks, since they don't provide nice error messages.
  stopifnot(is.data.frame(dat))
  stopifnot(is.na(x_label) || (is.character(x_label) && length(x_label) == 1))
  stopifnot(is.na(y_label) || (is.character(y_label) && length(y_label) == 1))
  
  # Convert passed parameters to strings, if they are not passed as strings.
  x_parm <- enquo(x_parm) # Read this in evaluated
  y_parm <- enquo(y_parm)
  color_parm <- enquo(color_parm)
  
  x_parmname <-  rlang::as_name(x_parm)
  y_parmname <-  rlang::as_name(y_parm)
  colorname  <-  rlang::as_name(color_parm)
  stopifnot(x_parmname %in% names(dat))
  stopifnot(y_parmname %in% names(dat)) 
  stopifnot(colorname %in% names(dat)) 

  y  <- dat[[y_parmname]]
  x  <- dat[[x_parmname]]
  color <- dat[[colorname]]
  
  plt <- ggplot(dat, aes(x, y)) +
    geom_point(aes(color = color), alpha = alpha, size = size) +
    xlab('') +
    ylab(y_parmname) +

    
    theme_cbep(base_size = 12) +
    theme(legend.position =  'bottom',
          legend.title = element_text(size = 10)) +
    
    guides(color = guide_legend(override.aes = list(alpha = 1, size = 2),
                                 byrow = TRUE)) +
  
  if(add_smooth) {
         plt <- plt + geom_smooth(method = 'gam', 
                                  formula = y ~ s(x),
                                  se = with_se,
                                  color = cbep_colors()[1])
         }
  
  if (is.factor(color)) {
    plt <- plt +
        scale_color_viridis_d(name = color_label, option = 'viridis')
  }
  else {
    plt <- plt +
        scale_color_viridis_c(name = color_label, option = 'viridis')
  }

  if(! is.na(x_label)) {
         plt <- plt + xlab(x_label)
  } 
  
  if(! is.na(y_label)) {
         plt <- plt + ylab(y_label)
  }
  return(plt)
}
```

### Helper Function to Add Medians by Color Groups

This function is intended principally to add summary points and lines to
cross plots – usually by month to emphasize seasonal patterns.

``` r
add_sum <- function(p, dat, x_parm, y_parm, color_parm = "Month", 
                    color_label = '', with_line = TRUE) {
  # These are ugly argument checks, since they don't provide nice error messages.
  stopifnot(is.data.frame(dat))
  
  # Convert passed parameters to strings, if they are not passed as strings.
  x_parm <- enquo(x_parm) # Read this in evaluated
  y_parm <- enquo(y_parm)
  color_parm <- enquo(color_parm)
  
  x_parmname <-  rlang::as_name(x_parm)
  y_parmname <-  rlang::as_name(y_parm)
  colorname  <-  rlang::as_name(color_parm)
  
  stopifnot(x_parmname %in% names(dat))
  stopifnot(y_parmname %in% names(dat)) 
  stopifnot(colorname %in% names(dat)) 

  df <- tibble(x     = dat[[x_parmname]],
               y     = dat[[y_parmname]],
               color = dat[[colorname]]) %>%
    group_by(color) %>%
    summarize(x = median(x, na.rm = TRUE),
              y = median(y, na.rm = TRUE)) %>%
    mutate(color = factor (color, levels = levels(color)))
  
  pp <- p
  
  if(with_line) {
    pp <- pp + geom_path(data = df, mapping = aes(x = x, y = y), size = 1,
              color = 'gray20')
  } 
  
  add <- pp + geom_point(data = df, mapping = aes(x = x, y = y, fill = color),
                        shape = 22, size = 3) +
          scale_fill_viridis_d(name = color_label, option = 'viridis')
    
  return(add)
}
```

# Values by Time

[Back to top](#)  
\#\# Example: Chlorophyll A

``` r
plt <- full_profile(the_data, chl_log1, dt = 'dt', color = 'Month', 
             label = 'Chlorophyll A (mg/l)', 
             guide = quantile(the_data$chl_log1, .9, na.rm = TRUE))
plt +
  scale_y_continuous(trans = 'log1p')
#> Warning: Removed 681 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/plt_chl_log1_prof-1.png" style="display: block; margin: auto;" />

## Faceted Plot

### Guide Functions to Transform Chlorophyll Axis

``` r
prefered_breaks =  c(0, 1, 5, 10, 50, 100)

my_breaks_fxn <- function(lims) {
  #browser()
  if(max(lims) < 10) {
    # Then we're looking at our transformed Chl data
  a <- prefered_breaks
    return(log(a +1))
  }
  else {
    return(labeling::extended(lims[[1]], lims[[2]], 5))
  }
}

my_label_fxn <- function(brks) {
  #browser()
  # frequently, brks is passed with NA in place of one or more 
  # of the candidate brks, even after I pass a vector of breaks.
  # In particular, "pretty" breaks outside the range of the data
  # are dropped and replaced with NA.
  a <- prefered_breaks
  b <- round(log(a+1), 3)
  real_breaks = round(brks[! is.na(brks)], 3)
  if (all(real_breaks %in% b)) {
    # then we have our transformed Chl data
    return(a)
  }
  else {
    return(brks)
  }
}
```

#### Plot

``` r
facet.labs <-         c("Temperature (°C)", 'Salinity (PSU)', 'Dissolved Oxygen (mg/l)', 
                        'Chlorophyll A (mg/l)')
names(facet.labs) <-  c("temperature", 'salinity', 'do', 'chl_log1')


long_data %>%
  filter(Parameter %in% c("temperature", 'salinity', 'do', 'chl_log1')) %>%
  mutate(Parameter = factor(Parameter, 
                            levels =  c("temperature", 'salinity', 'do', 'chl_log1'))) %>%
  mutate(Parameter4 = factor(Parameter, 
                        levels =  c("temperature", 'salinity', 
                                        'do', 'chl_log1'),
                        labels = c(expression("Temperature (" * degree * "C)"), 
                                  expression('Salinity' ~ '(PSU)'), 
                                  expression('Dissolved' ~ 'Oxygen' ~ '(mg/l)'),
                                  expression('Chlorophyll A (' * mu * 'g/l)')))) %>%
  full_profile(parm = Value, dt = 'dt', color = 'hour', alpha = 1,
               label = '', color_label = 'Hour of the Day') +
  #geom_hline(data = href, mapping = aes(yintercept = val),
  #           lty = 'dotted', color = 'gray15') +
  
  
  scale_y_continuous (breaks = my_breaks_fxn, labels = my_label_fxn) +
   # scale_color_viridis_c(name = 'Time of Day',
   #                       option = 'viridis', 
   #                       breaks = c(0,3,6,9,12,15,18,21,24)) +
  
  scale_color_continuous_diverging(palette = 'Cork', 
                                   mid = 12,
                                   breaks = c(0,3,6, 9, 12,15,18, 21),
                                   name = 'Hour of the Day') +
  
  facet_wrap(~ Parameter4, nrow = 2, scales = 'free_y',
             labeller = label_parsed)
#> Scale for 'colour' is already present. Adding another scale for 'colour',
#> which will replace the existing scale.
#> Warning: Removed 2745 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/facet_prof-1.png" style="display: block; margin: auto;" />

``` r

 ggsave('figures/cms_history.pdf', device = cairo_pdf, 
        width = 7, height = 5)
#> Warning: Removed 2745 rows containing missing values (geom_point).
```

# Seasonal Profiles

[Back to top](#)  
These graphs combine data from multiple years to generate a picture of
seasonal patterns. Since data coverage is inconsistent year to year,
data for some times of year are derived from just one or two years,
which could bias the results.

## Example: Chlorophyll A

``` r
season_profile(the_data, chl_log1, doy, alpha = 0.5,
               size = .5,
               label = "Chlorophyll A (mg/l)",
               add_smooth = TRUE, 
               with_se = FALSE) +
  scale_y_continuous(trans = 'log1p')
#> Warning: Removed 681 rows containing non-finite values (stat_smooth).
#> Warning: Removed 681 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/climatology_chl-1.png" style="display: block; margin: auto;" />

## Joint Plot

``` r
long_data %>%
  filter(Parameter %in% c("temperature", 'salinity', 'do', 'chl_log1')) %>%
  mutate(Parameter = factor(Parameter, 
                            levels =  c("temperature", 'salinity', 'do', 'chl_log1'))) %>%
  mutate(Parameter4 = factor(Parameter, 
                            levels =  c("temperature", 'salinity', 'do', 'chl_log1'),
                            labels = c(expression("Temperature (" * degree * "C)"), 
                                       expression('Salinity' ~ '(PSU)'), 
                                       expression('Dissolved' ~ 'Oxygen' ~ '(mg/l)'),
                                       expression('Chlorophyll A (' * mu * 'g/l)')))) %>%
  season_profile(Value, doy = 'doy',
                size = .5, alpha = 0.25,
                label = '',
                add_smooth = FALSE, 
                 with_se = FALSE) +

  
  scale_y_continuous (breaks = my_breaks_fxn, labels = my_label_fxn) +
  
  facet_wrap(~ Parameter4, nrow = 2, scales = 'free_y',
             labeller = label_parsed)
#> Warning: Removed 2745 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/facet_climateology-1.png" style="display: block; margin: auto;" />

``` r
  
 ggsave('figures/cms_climatology.pdf', device = cairo_pdf, 
        width = 7, height = 5)
#> Warning: Removed 2745 rows containing missing values (geom_point).
```

## Revised Plot

``` r
# Flexible labeling for months of the year
monthlengths <-  c(31,28,31, 30,31,30,31,31,30,31,30,31)
cutpoints    <- c(0, cumsum(monthlengths)[1:12])
monthlabs    <- c(month.abb,'')

plt <-  long_data %>%
  filter(Parameter %in% c("temperature", 'salinity', 'do', 'chl_log1')) %>%
  mutate(Parameter = factor(Parameter, 
                            levels =  c("temperature", 'salinity', 
                                        'do', 'chl_log1'))) %>%
  mutate(Parameter4 = factor(Parameter, 
                             levels =  c("temperature", 'salinity',
                                         'do', 'chl_log1'),
                             labels = c(expression("Temperature (" * degree * "C)"), 
                                        expression('Salinity' ~ '(PSU)'), 
                                        expression('Dissolved' ~ 'Oxygen' ~ '(mg/l)'),
                                        expression('Chlorophyll A (' * mu * 'g/l)')))) %>%
  
  ggplot(aes(doy, Value)) +
  geom_point(aes(color = factor(year)), alpha = 0.25, size = 0.5) +
  xlab('') +
  ylab('') +
  
  scale_color_manual(values=cbep_colors()[c(3,2,5,4,6)], name='Year') +
  scale_x_continuous(breaks = cutpoints, labels = monthlabs) +
  scale_y_continuous (breaks = my_breaks_fxn, labels = my_label_fxn) +
  
  theme_cbep(base_size = 12) +
  theme(axis.text.x=element_text(angle=90, vjust = 1.5)) +
  theme(legend.position =  'bottom',
        legend.title = element_text(size = 10)) +
  
  facet_wrap(~ Parameter4, nrow = 2, scales = 'free_y',
             labeller = label_parsed) +
  guides(color = guide_legend(override.aes = list(alpha = 1, size = 4)))
plt
#> Warning: Removed 2745 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/facet_climatology_direct-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/cms_climatology_revised.pdf', device = cairo_pdf, 
        width = 7, height = 5)
#> Warning: Removed 2745 rows containing missing values (geom_point).
```

# Cross-Plots

[Back to top](#)  
\#\# Dissolved Oxygen and PCO<sub>2</sub>

``` r
plt <- cross_plot(the_data, do, pco2, color_parm = "Month",
                            x_label = "Dissolved Oxygen (mg/l)", 
                            y_label = "pCO2 (uAtm)",
                            alpha = 0.1,
                            size = 0.5,
                            add_smooth = FALSE,
                            with_se = FALSE)
plt <- plt + 
  ylab(expression (pCO[2]~(mu*Atm)))+
  guides(color = guide_legend(override.aes = list(alpha = 1, size = 3),
                              byrow = TRUE,  nrow = 2))
plt
#> Warning: Removed 10896 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/cross_plot_do_co2-1.png" style="display: block; margin: auto;" />

``` r
add_sum (plt, the_data, do, pco2, with_line = TRUE)
#> Warning: Removed 10896 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/cross_plot_do_co2_sum-1.png" style="display: block; margin: auto;" />

## Dissolved Oxygen and Temperature

``` r
plt <- cross_plot(the_data, temperature, do, color_parm = "Month",
                            y_label = "Dissolved Oxygen (mg/l)", 
                            x_label = "Temperature (°C)",
                            alpha = 0.1,
                            size = 0.5,
                            add_smooth = FALSE,
                            with_se = FALSE) +
  guides(color = guide_legend(override.aes = list(alpha = 1, size = 3),
                              byrow = TRUE,  nrow = 2))
plt
#> Warning: Removed 912 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/cross_plot_do_t-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/cms_do_temp.pdf', device = cairo_pdf, 
        width = 5, height = 4)
#> Warning: Removed 912 rows containing missing values (geom_point).
```

### Add Mpnthly Medians

``` r
plt <- cross_plot(the_data, temperature, do, color_parm = "Month",
                            y_label = "Dissolved Oxygen (mg/l)", 
                            x_label = "Temperature (°C)",
                            alpha = 0.1,
                            size = 0.5,
                            add_smooth = FALSE,
                            with_se = FALSE)

add_sum (plt, the_data, temperature, do, with_line = TRUE) +
    guides(color = guide_legend(override.aes = list(alpha = 1, size = 3),
                              byrow = TRUE,  nrow = 2))
#> Warning: Removed 912 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/cross_plot_do_t_2-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/cms_do_temp_2.pdf', device = cairo_pdf, 
        width = 5, height = 4)
#> Warning: Removed 912 rows containing missing values (geom_point).
```

### Compare Daily Data

``` r
plt <- cross_plot(daily_data, temperature_med, do_med, color_parm = "Month",
                            y_label = "Dissolved Oxygen (mg/l)", 
                            x_label = "Temperature (°C)",
                            alpha = 1,
                            size = 01,
                            add_smooth = FALSE,
                            with_se = FALSE)

add_sum (plt, daily_data, temperature_med, do_med, with_line = TRUE) +
    guides(color = guide_legend(override.aes = list(alpha = 1, size = 3),
                              byrow = TRUE,  nrow = 2))
#> Warning: Removed 27 rows containing missing values (geom_point).
```

<img src="FOCB_CMS1_graphics_Summary_files/figure-gfm/cross_plot_do_co2_daily-1.png" style="display: block; margin: auto;" />

ggsave(‘figures/cms\_climatology\_revised.pdf’, device = cairo\_pdf,
width = 7, height = 5) \`\`\`
