
Interpolate a time series of polygons with areal weighting through
tidyr::nest and purrr::map.

# Packages

``` r
library(tidyverse)
library(viridis)
library(sf)
library(reprex)
```

# Data

Package sf includes sample data described at
<https://r-spatial.github.io/spdep/articles/sids.html>.

``` r
demo(nc, 
     ask = FALSE, 
     echo = FALSE)
```

    ## Reading layer `nc.gpkg' from data source `C:\Users\steinkruger\Documents\R\win-library\3.6\sf\gpkg\nc.gpkg' using driver `GPKG'
    ## Simple feature collection with 100 features and 14 fields
    ## Attribute-geometry relationship: 0 constant, 8 aggregate, 6 identity
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: -84.32385 ymin: 33.88199 xmax: -75.45698 ymax: 36.58965
    ## epsg (SRID):    4267
    ## proj4string:    +proj=longlat +datum=NAD27 +no_defs

# Wrangle

Wrangle the North Carolina SIDS dataset to (a) a time series of
intensive data and (b) a reference grid.

``` r
# (a)
dat = 
  nc %>% 
  mutate(PNWBIR74 = NWBIR74 / BIR74,
         PNWBIR79 = NWBIR79 / BIR79) %>% # Calculate intensive columns (%).
  select(PNWBIR74,
         PNWBIR79) %>% # Cut extensive columns.
  data.frame %>% # Switch geometries to non-sf column for pivot_longer.
  pivot_longer(cols = c(PNWBIR74,
                        PNWBIR79),
               names_to = "year",
               values_to = "perc") %>% # Pivot for consistency w/ use case.
  mutate(year = ifelse(year == "PNWBIR74",
                       1974,
                       1979)) %>% # Get years into numeric.
  data.frame %>% # Get class back to data.frame. This doesn't matter.
  st_sf %>% # Get geometries back to sf.
  rename(geometry = geom) # Fix geometries name for rbind w/ interpolation output.

# (b)
ref = 
  nc %>% 
  st_make_grid(n = c(20, 
                     10)) # Get a grid for interpolation.
```

# Run

Interpolate the time series of intensive data onto the reference grid,
with (b) and without (a) explicit time.

``` r
# Get results w/o accounting for years.
out_wrong = 
  dat %>% 
  st_interpolate_aw(to = ref,
                    extensive = FALSE) %>% 
  mutate(year = "(Interpolated)") %>% 
  select(-Group.1) %>% 
  rbind(dat)

# Get results w/ explicit years. 
out_right = 
  dat %>% 
  group_by(year) %>% 
  nest %>% # Turn data into a list column by year.
  mutate(Output = 
           data %>% 
           map(.f = st_interpolate_aw,
               to = ref,
               extensive = FALSE) %>% # Interpolate data to the reference grid.
           map(.f = select,
               -Group.1), # Tag on explicit identifier for output.
         Input = 
           data %>% 
           map(.x = .,
               .f = data.frame)) %>% 
  select(-data) %>% 
  pivot_longer(cols = c("Input",
                        "Output"),
               names_to = "names",
               values_to = "data") %>% 
  unnest %>% # Get data back out of the list column.
  ungroup %>% 
  data.frame %>% 
  rbind %>% 
  st_sf
```

# Visualize

Illustrate the input and output without explicit years.

``` r
ggplot() + 
  geom_sf(data = out_wrong, 
          aes(fill = perc), 
          color = NA) + 
  geom_sf(data = ref,
          fill = NA,
          color = "grey50",
          size = 0.25) +
  facet_wrap(~ year,
             ncol = 1) + 
  scale_fill_viridis() + 
  theme_void() +
  theme(legend.position = "none")
```

![](reprex_map_interpolate_files/figure-gfm/vis_wrong-1.png)<!-- -->

Illustrate the input and output with explicit years.

``` r
ggplot() + 
  geom_sf(data = out_right, 
          aes(fill = perc), 
          color = NA) + 
  geom_sf(data = ref,
          fill = NA,
          color = "grey50",
          size = 0.25) +
  facet_wrap(year ~ names) + 
  scale_fill_viridis() + 
  theme_void() +
  theme(legend.position = "none")
```

![](reprex_map_interpolate_files/figure-gfm/vis_right-1.png)<!-- -->
