
# Using `viridis'` “inferno” colour scheme to map Australian temperatures on Feb. 11, 2017

This is a fork of [njtierney’s](https://github.com/njtierney/)
[ozviridis](https://github.com/njtierney/ozviridis) project. I’m
building on what he’s started, but making the map my way.

As Nicholas originally observed on 12-Feb-2017,

> "there’s a heatwave in Australia at the moment. And this is the
> heatmap that is getting shown of Australia:

<img src="bom-heat-map.png" width="50%" style="display: block; margin: auto;" />

> Which shows that things are really hot.

Verymuchso.

> But it’s also pretty darn ugly.

Almost as ugly as the temperatures outside.

Nicholas wanted to see if he could use the `viridis` package to improve
this map. I completely agree. This should be doable.

## Setup

To do this you’ll need a few packages from CRAN:

``` r
library(raster)
```

    ## Loading required package: sp

``` r
library(ggplot2)
library(viridis)
```

    ## Loading required package: viridisLite

``` r
library(ggthemes)
library(grid)
library(gridExtra)
```

One from ROpenSciLabs,
[rnaturalearth](https://github.com/ropenscilabs/rnaturalearth):

``` r
#if (!require("devtools")) install.packages("devtools")
#devtools::install_github("ropenscilabs/rnaturalearth")
library("rnaturalearth")
```

And one from SWISH,
[awaptools](https://github.com/swish-climate-impact-assessment/awaptools):

``` r
#devtools::install_github("swish-climate-impact-assessment/awaptools")
library(awaptools)
```

## Get the data to recreate our map

### BoM AWAP grids

Using `awaptools` get the mean maximum for February 11 2017.

``` r
awaptools::get_awap_data(start = "2017-02-11", end = "2017-02-11", measure_i = "maxave")

oz_heat <- raster::raster("maxave_2017021120170211.grid")
```

``` r
raster::plot(oz_heat, col = heat.colors(n = length(unique(oz_heat))))
```

![](README_files/figure-gfm/plot_bom_grid-1.png)<!-- -->

The plot works, the colours might not be any better than BoM though.
We’ll get to that.

### Adding a shapefile of Australia

Now that the temperature data is in R, use the `rnaturalearthdata`
package to get an outline of Australia. We’ll use this to map so we can
see the states, but also to clean up the map a bit, see that previous
figure? You can sorta pick Australia out, but it’s not clearly defined.
We can use this to fix that.

``` r
oz_shape <- rnaturalearth::ne_states(geounit = "australia")

sp::plot(oz_shape)
```

![](README_files/figure-gfm/australia-1.png)<!-- -->

### Clean up the heat map

Using the [Naturalearthdata](http://www.naturalearthdata.com) object,
now mask out only landmasses and trim down the outline, removing islands
that stretch the map and aren’t of interest. Note that we mask using the
naturalearth object and crop using the heat map.

``` r
oz_heat <- raster::mask(oz_heat, oz_shape)
oz_shape <- raster::crop(oz_shape, oz_heat)
```

    ## Loading required namespace: rgeos

# Plot using `ggplot2` and `viridis`

Now we’re ready to plot this up using `ggplot2`, but first, we need to
make the raster object into a format that `ggplot2` can use.

``` r
# Extract the data into a data.frame for ggplot2
oz_heat_df <- as.data.frame(raster::rasterToPoints(oz_heat))

# Make appropriate column headings
colnames(oz_heat_df) <- c("Longitude", "Latitude", "Temperature")
```

## Classify the heat map

BoM shows the map in 3˚C increments. We can reclassify the raster so
that it will display in the same way.

Using the `cut` function, we’ll set up our map in the same way.

``` r
oz_heat_df$cuts <- as.factor(cut(oz_heat_df$Temperature,
                           include.lowest = TRUE,
                           breaks = seq(-6, 54, by = 3)))
```

## The final product

Now, you can plot these together. Plot the new `data.frame`, `oz_heat`
and layer a map of Australia on top of it. The Australian map can be
plotted directly from the spatial data, using the following method as
described in the tidyverse here,
<https://github.com/tidyverse/ggplot2/wiki/plotting-polygon-shapefiles>:

``` r
oz <- ggplot2::ggplot(data = na.omit(oz_heat_df), 
                      aes(y = Latitude, x = Longitude)) +
  ggplot2::geom_raster(aes(fill = cuts)) +
  viridis::scale_fill_viridis(option = "inferno", discrete = TRUE) +
  ggplot2::guides(fill = guide_legend(reverse = TRUE)) +
  ggplot2::geom_polygon(data = oz_shape, 
                        aes(x = long, y = lat, group = group),
                        fill = NA, color = "black", size = 0.25) +
  ggthemes::theme_map() +
  ggplot2::theme(legend.position = c(1, 0.15),
                 legend.text = element_text(size = 8),
                 legend.title = element_blank()) +
  ggplot2::labs(title = "Maximum Temperature (˚C)", 
                subtitle = "11th February, 2017", 
                caption = "Data: Australia Bureau of Meteorology (AWAP) and Naturalearthdata") +
  ggplot2::coord_quickmap()
```

    ## Regions defined for each Polygons

``` r
# Using the gridExtra and grid packages add a neatline to the map
gridExtra::grid.arrange(oz, ncol = 1)
grid::grid.rect(width = 0.98, 
                height = 0.98, 
                gp = grid::gpar(lwd = 0.25, 
                                col = "black",
                                fill = NA))
```

![](README_files/figure-gfm/plot-1.png)<!-- -->

That’s much better and pretty close to what BoM originally created.
Using any of the `ggplot` `panel.background` or `panel.grid` result in
the legend being outside the line, so not really a neatline for a map.
Using the `gridExtra` and `grid` packages fixes this.

## Cleanup on the way out

Remove the grid file that we downloaded earlier.

``` r
unlink("maxave_2017021120170211.grid")
```

# Meta

Please note that this project is released with a [Contributor Code of
Conduct](CONDUCT.md). By participating in this project you agree to
abide by its terms.

# Appendix

    ## R version 3.5.3 (2019-03-11)
    ## Platform: x86_64-apple-darwin15.6.0 (64-bit)
    ## Running under: macOS Mojave 10.14.4
    ## 
    ## Matrix products: default
    ## BLAS: /Library/Frameworks/R.framework/Versions/3.5/Resources/lib/libRblas.0.dylib
    ## LAPACK: /Library/Frameworks/R.framework/Versions/3.5/Resources/lib/libRlapack.dylib
    ## 
    ## locale:
    ## [1] en_AU.UTF-8/en_AU.UTF-8/en_AU.UTF-8/C/en_AU.UTF-8/en_AU.UTF-8
    ## 
    ## attached base packages:
    ## [1] grid      stats     graphics  grDevices utils     datasets  methods  
    ## [8] base     
    ## 
    ## other attached packages:
    ## [1] awaptools_1.2.1     rnaturalearth_0.1.0 gridExtra_2.3      
    ## [4] ggthemes_4.1.1      viridis_0.5.1       viridisLite_0.3.0  
    ## [7] ggplot2_3.1.1       raster_2.8-19       sp_1.3-1           
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] Rcpp_1.0.1               pillar_1.3.1            
    ##  [3] compiler_3.5.3           plyr_1.8.4              
    ##  [5] class_7.3-15             tools_3.5.3             
    ##  [7] digest_0.6.18            evaluate_0.13           
    ##  [9] tibble_2.1.1             gtable_0.3.0            
    ## [11] lattice_0.20-38          pkgconfig_2.0.2         
    ## [13] rlang_0.3.4              DBI_1.0.0               
    ## [15] rgdal_1.4-3              yaml_2.2.0              
    ## [17] xfun_0.6                 e1071_1.7-1             
    ## [19] withr_2.1.2              stringr_1.4.0           
    ## [21] dplyr_0.8.0.1            knitr_1.22              
    ## [23] rgeos_0.4-2              classInt_0.3-1          
    ## [25] tidyselect_0.2.5         glue_1.3.1              
    ## [27] sf_0.7-3                 R6_2.4.0                
    ## [29] rmarkdown_1.12           purrr_0.3.2             
    ## [31] magrittr_1.5             rnaturalearthhires_0.2.0
    ## [33] units_0.6-2              scales_1.0.0            
    ## [35] codetools_0.2-16         htmltools_0.3.6         
    ## [37] assertthat_0.2.1         colorspace_1.4-1        
    ## [39] labeling_0.3             stringi_1.4.3           
    ## [41] lazyeval_0.2.2           munsell_0.5.0           
    ## [43] crayon_1.3.4
