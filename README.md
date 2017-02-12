
This is a fork of @njtierney's [ozviridis](https://github.com/njtierney/ozviridis) project. I'm building on what he's started, but making the map my way.

As Nicholas originally observed on 12-Feb-2017,

> "there's a heatwave in Australia at the moment. And this is the heatmap that is getting shown of Australia:

<img src="bom-heat-map.png" width="50%" style="display: block; margin: auto;" />

> Which shows that things are really hot.

Verymuchso.

> But it's also pretty darn ugly.

Almost as ugly as the temperatures outside.

Nicholas wanted to see if he could use the `viridis` package to improve this map. I completely agree. This should be doable.

To do this you only need a few packages from CRAN:

``` r
library(raster)
```

    ## Loading required package: sp

``` r
library(ggplot2)
library(viridis)
library(ggthemes)
```

and one from ROpenSciLabs, [rnaturalearth](https://github.com/ropenscilabs/rnaturalearth):

``` r
# if (!require("devtools")) install.packages("devtools")
# devtools::install_github("ropenscilabs/rnaturalearth")

library("rnaturalearth")
```

Nicholas already found where to get the data from [BoM]((http://www.bom.gov.au/jsp/awap/temp/index.jsp)), courtesy of [Robbi Bishop Taylor](https://twitter.com/robbibt). Download the grid file version, uncompress it and import it using raster.

``` r
oz_heat <- raster::raster("~/Downloads/latest.grid")
raster::plot(oz_heat)
```

![](README_files/figure-markdown_github/read_grid-1.png)

The plot works, the colours might not be any better than BoM though. We'll get to that.

Adding the shapefile of Australia
---------------------------------

Now that the temperature data is in R, use the `rnaturalearthdata` package to get an outline of Australia. We'll use this to map so we can see the states, but also to clean up the map a bit, see that previous figure? You can sorta pick Australia out, but it's not clearly defined. We can use this to fix that.

``` r
oz_shape <- rnaturalearth::ne_states(geounit = "australia")

sp::plot(oz_shape)
```

![](README_files/figure-markdown_github/australia-1.png)

Clean up the heat map
---------------------

Using the Naturalearthdata object, now mask out only landmasses and trim down the outline, removing islands that stretch the map and aren't of interest. Note that we mask using the naturalearth object and crop using the heat map.

``` r
oz_heat <- mask(oz_heat, oz_shape)
oz_shape <- crop(oz_shape, oz_heat)
```

    ## Loading required namespace: rgeos

Plot using ggplot2
------------------

Now we're ready to plot this up using `ggplot2`, but first, we need to make the raster object into a format that `ggplot2` can use.

``` r
# Extract the data into a matrix
oz_heat <- rasterToPoints(oz_heat)

# Make the matrix a dataframe for ggplot
oz_heat_df <- data.frame(oz_heat)

# Make appropriate column headings
colnames(oz_heat_df) <- c("Longitude", "Latitude", "Temperature")
```

Now, you can plot this directly from the spatial data, using the following method as described in the tidyverse here, <https://github.com/tidyverse/ggplot2/wiki/plotting-polygon-shapefiles>:

``` r
ggplot(data = oz_heat_df, aes(y = Latitude, x = Longitude)) +
  geom_raster(aes(fill = Temperature)) +
  scale_fill_viridis(option = "inferno") +
  geom_polygon(data = oz_shape, aes(x = long, y = lat, group = group),
               fill = NA, color = "black", size = 0.25) +
  theme_map() +
  theme(legend.position = c(1, .5)) +
  coord_quickmap()
```

    ## Regions defined for each Polygons

![](README_files/figure-markdown_github/plot-1.png)
