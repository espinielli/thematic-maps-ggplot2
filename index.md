<img src="/wp-content/uploads/2016/12/tm-final-map-1.png" width="100%" />

The above *choropleth* was created with `ggplot2` (2.2.0) only. Well,
almost. Of course, you need the usual suspects such as `rgdal` and
`rgeos` when dealing with geodata, and `raster` for the relief. But
apart from that: nothing fancy such as `ggmap` or the like. The imported
packages are kept to an absolute minimum.

In this blog post, I am going to explain step by step how I (eventually)
achieved this result – from a very basic, useless, ugly, default map to
the publication-ready and (in my opinion) highly aesthetic choropleth.

-   [Reproducibility](#reproducibility)
-   [Preparations](#preparations)
    -   [Clear workspace and install necessary
        packages](#clear-workspace-and-install-necessary-packages)
    -   [General ggplot2 theme for
        map](#general-ggplot2-theme-for-map)
-   [Data sources](#data-sources)
    -   [Read in data and preprocess](#read-in-data-and-preprocess)
    -   [Read in geodata](#read-in-geodata)
-   [A very basic map](#a-very-basic-map)
    -   [A better color scale](#a-better-color-scale)
    -   [Horizontal legend](#horizontal-legend)
-   [Discrete classes with quantile
    scale](#discrete-classes-with-quantile-scale)
    -   [Discrete classes with pretty
        breaks](#discrete-classes-with-pretty-breaks)
    -   [More intuitive legend](#more-intuitive-legend)
    -   [Better colors for classes](#better-colors-for-classes)
-   [Relief](#relief)
-   [Final map](#final-map)
-   [Update, January 2nd, 2017](#update-january-2nd-2017)

Reproducibility {#reproducibility}
---------------

As always, you can reproduce, reuse and remix everything you find here,
just go to [this
repository](https://github.com/grssnbchr/thematic-maps-ggplot2) and
clone it. All the needed input files are in the `input` folder, and the
main file to execute is `index.Rmd`. Right now, knitting it produces an
`index.md` that I use for my blog post on
[timogrossenbacher.ch](https://timogrossenbacher.ch), but you can adapt
the script to produce an HTML file, too. The PNGs produced herein are
saved to `/wp-content/uploads/2016/12` so I can display them directly in
my blog, but of course you can also adjust this.

Preparations {#preparations}
------------

### Clear workspace and install necessary packages {#clear-workspace-and-install-necessary-packages}

This is just my usual routine: Detach all packages, remove all variables
in the global environment, etc, and then load the packages. Saves me a
lot of headaches.

``` r
knitr::opts_chunk$set(
    out.width = "100%",
    dpi = 300,
    fig.width = 8,
    fig.height = 6,
    fig.path = '/wp-content/uploads/2016/12/tm-',
    strip.white = T,
    dev = "png",
    dev.args = list(png = list(bg = "transparent"))
)

remove(list = ls(all.names = TRUE))

detachAllPackages <- function() {
  basic.packages.blank <-  c("stats", 
                             "graphics", 
                             "grDevices", 
                             "utils", 
                             "datasets", 
                             "methods", 
                             "base")
  basic.packages <- paste("package:", basic.packages.blank, sep = "")
  
  package.list <- search()[ifelse(unlist(gregexpr("package:", search())) == 1, 
                                  TRUE, 
                                  FALSE)]
  
  package.list <- setdiff(package.list, basic.packages)
  
  if (length(package.list) > 0)  for (package in package.list) {
    detach(package, character.only = TRUE)
    print(paste("package ", package, " detached", sep = ""))
  }
}

detachAllPackages()


if (!require(rgeos)) {
  install.packages("rgeos", repos = "http://cran.us.r-project.org")
  require(rgeos)
}
if (!require(rgdal)) {
  install.packages("rgdal", repos = "http://cran.us.r-project.org")
  require(rgdal)
}
if (!require(raster)) {
  install.packages("raster", repos = "http://cran.us.r-project.org")
  require(raster)
}
if(!require(ggplot2)) {
  install.packages("ggplot2", repos="http://cloud.r-project.org")
  require(ggplot2)
}
if(!require(viridis)) {
  install.packages("viridis", repos="http://cloud.r-project.org")
  require(viridis)
}
if(!require(dplyr)) {
  install.packages("dplyr", repos = "https://cloud.r-project.org/")
  require(dplyr)
}
if(!require(gtable)) {
  install.packages("gtable", repos = "https://cloud.r-project.org/")
  require(gtable)
}
if(!require(grid)) {
  install.packages("grid", repos = "https://cloud.r-project.org/")
  require(grid)
}
```

### General ggplot2 theme for map {#general-ggplot2-theme-for-map}

First of all, I define a generic theme that will be used as the basis
for all of the following steps. It's based on `theme_minimal` and
basically resets all the axes. It also defined a very subtle grid and a
warmgrey background, which gives it some sort of paper map feeling, I
find.

The font used here is `Ubuntu Regular` – adapt to your liking, but the
font must be installed on your OS.

``` r
theme_map <- function(...) {
  theme_minimal() +
  theme(
    text = element_text(family = "Ubuntu Regular", color = "#22211d"),
    axis.line = element_blank(),
    axis.text.x = element_blank(),
    axis.text.y = element_blank(),
    axis.ticks = element_blank(),
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    # panel.grid.minor = element_line(color = "#ebebe5", size = 0.2),
    panel.grid.major = element_line(color = "#ebebe5", size = 0.2),
    panel.grid.minor = element_blank(),
    plot.background = element_rect(fill = "#f5f5f2", color = NA), 
    panel.background = element_rect(fill = "#f5f5f2", color = NA), 
    legend.background = element_rect(fill = "#f5f5f2", color = NA),
    panel.border = element_blank(),
    ...
  )
}
```

Data sources {#data-sources}
------------

For this choropleth, I used **three** data sources:

-   Thematic data: Average age per municipality as of end of 2015. The
    data is freely available from [The Swiss Federal Statistical
    Office (FSO)](https://www.bfs.admin.ch/bfs/de/home/statistiken/bevoelkerung/stand-entwicklung/alter-zivilstand-staatsangehoerigkeit.html)
    and included in the `input` folder.
-   Municipality geometries: The geometries do not show the political
    borders of Swiss municipalities, but the so-called "productive"
    area, i.e., larger lakes and other "unproductive" areas such as
    mountains are excluded. This has two advantages: 1) The relatively
    sparsely populated but very large municipalities in the Alps don't
    have too much visual weight and 2) it allows us to use the beautiful
    raster relief of the Alps as a background. The data are also from
    the FSO, but not freely available. You could also use the freely
    available [political
    boundaries](https://www.bfs.admin.ch/bfs/de/home/dienstleistungen/geostat/geodaten-bundesstatistik/administrative-grenzen/generalisierte-gemeindegrenzen.html)
    of course. I was allowed to republish the Shapefile for this
    educational purpose (also included in the `input` folder). Please
    stick to that policy.
-   Relief: This is a freely available GeoTIFF from [The Swiss Federal
    Office of
    Topography (swisstopo)](https://shop.swisstopo.admin.ch/en/products/maps/national/digital/srm1000).

### Read in data and preprocess {#read-in-data-and-preprocess}

``` r
data <- read.csv("input/avg_age_15.csv", stringsAsFactors = F)
```

### Read in geodata {#read-in-geodata}

Here, the geodata is loaded using `rgeos` / `rgdal` standard procedures.
It is then *"fortified"*, i.e. transformed into a ggplot2-compatible
data frame (the `fortify`-function is part of `ggplot2`). Also, the
thematic data is joined using the `bfs_id` field (each municipality has
a unique one).

``` r
gde_15 <- readOGR("input/geodata/gde-1-1-15.shp", layer = "gde-1-1-15")
```

    ## OGR data source with driver: ESRI Shapefile 
    ## Source: "input/geodata/gde-1-1-15.shp", layer: "gde-1-1-15"
    ## with 2324 features
    ## It has 2 fields

``` r
# set crs to ch1903/lv03, just to make sure  (EPSG:21781)
crs(gde_15) <- "+proj=somerc +lat_0=46.95240555555556 
+lon_0=7.439583333333333 +k_0=1 +x_0=600000 +y_0=200000 
+ellps=bessel +towgs84=674.374,15.056,405.346,0,0,0,0 +units=m +no_defs"
# fortify, i.e., make ggplot2-compatible
map_data_fortified <- fortify(gde_15, region = "BFS_ID") %>% 
  mutate(id = as.numeric(id))
# now we join the thematic data
map_data <- map_data_fortified %>% left_join(data, by = c("id" = "bfs_id"))

# read in background relief
relief <- raster("input/geodata/02-relief-georef-clipped-resampled.tif")
relief_spdf <- as(relief, "SpatialPixelsDataFrame")
# relief is converted to a very simple data frame, 
# just as the fortified municipalities.
# for that we need to convert it to a 
# SpatialPixelsDataFrame first, and then extract its contents 
# using as.data.frame
relief <- as.data.frame(relief_spdf) %>% 
  rename(value = `X02.relief.georef.clipped.resampled`)
# remove unnecessary variables
rm(relief_spdf)
rm(gde_15)
rm(map_data_fortified)
```

A very basic map {#a-very-basic-map}
----------------

What follows now is a very basic map with the municipalities rendered
with `geom_polygon` and their outline with `geom_path`. I don't even
define a color scale here, it just uses ggplot2's default continuous
color scale, because `avg_age_15` is a continuous variable.

Because the geodata are in a projected format, it is important to use
`coord_equal()` here, if not, Switzerland would be distorted.

``` r
p <- ggplot() +
    # municipality polygons
    geom_polygon(data = map_data, aes(fill = avg_age_15, 
                                      x = long, 
                                      y = lat, 
                                      group = group)) +
    # municipality outline
    geom_path(data = map_data, aes(x = long, 
                                   y = lat, 
                                   group = group), 
              color = "white", size = 0.1) +
    coord_equal() +
    # add the previously defined basic theme
    theme_map() +
    labs(x = NULL, 
         y = NULL, 
         title = "Switzerland's regional demographics", 
         subtitle = "Average age in Swiss municipalities, 2015", 
         caption = "Geometries: ThemaKart, BFS; Data: BFS, 2016")
p
```

<img src="/wp-content/uploads/2016/12/tm-basic-map-1.png" width="100%" />

How ugly! The color scale is not very sensitive to the data at hand,
i.e., regional patterns cannot be detected at all.

### A better color scale {#a-better-color-scale}

See how I reuse the previously defined `p`-object and just add the
continuous `viridis` scale from the same named package. All of a sudden
the map looks more aesthetic and regional patterns are already visible
in this linear scale. For example one can see that the municipalities in
the south and in the Alps (where there are a lot of gaps, the
unproductive areas I talked about) seem to have an older-than-average
population (mainly because young people move to the cities for work
etc.).

``` r
q <- p + scale_fill_viridis(option = "magma", direction = -1)
q
```

<img src="/wp-content/uploads/2016/12/tm-basic-map-viridis-1.png" width="100%" />

### Horizontal legend {#horizontal-legend}

Also I think one could save some space by using a horizontal legend at
the bottom of the plot.

``` r
q <- p +
  # this is the main part
  theme(legend.position = "bottom") +
  scale_fill_viridis(
    option = "magma", 
    direction = -1,
    name = "Average age",
    # here we use guide_colourbar because it is still a continuous scale
    guide = guide_colorbar(
      direction = "horizontal",
      barheight = unit(2, units = "mm"),
      barwidth = unit(50, units = "mm"),
      draw.ulim = F,
      title.position = 'top',
      # some shifting around
      title.hjust = 0.5,
      label.hjust = 0.5
  ))
q
```

<img src="/wp-content/uploads/2016/12/tm-basic-map-viridis-horizontal-1.png" width="100%" />

Well, the plot now has a weird aspect ratio, but okay...

Discrete classes with quantile scale {#discrete-classes-with-quantile-scale}
------------------------------------

I am still not happy with the color scale because I think regional
patterns could be made more clearly visible. For that I break up the
continuous `avg_age_15` variable into 6 quantiles (remember your
statistics class?). The effect of that is that I now have about the same
number of municipalities in each class.

``` r
no_classes <- 6
labels <- c()

quantiles <- quantile(map_data$avg_age_15, 
                      probs = seq(0, 1, length.out = no_classes + 1))

# here I define custom labels (the default ones would be ugly)
labels <- c()
for(idx in 1:length(quantiles)){
  labels <- c(labels, paste0(round(quantiles[idx], 2), 
                             " – ", 
                             round(quantiles[idx + 1], 2)))
}
# I need to remove the last label 
# because that would be something like "66.62 - NA"
labels <- labels[1:length(labels)-1]

# here I actually create a new 
# variable on the dataset with the quantiles
map_data$avg_age_15_quantiles <- cut(map_data$avg_age_15, 
                                     breaks = quantiles, 
                                     labels = labels, 
                                     include.lowest = T)

p <- ggplot() +
    # municipality polygons (watch how I 
   # use the new variable for the fill aesthetic)
    geom_polygon(data = map_data, aes(fill = avg_age_15_quantiles, 
                                      x = long, 
                                      y = lat, 
                                      group = group)) +
    # municipality outline
    geom_path(data = map_data, aes(x = long, 
                                   y = lat, 
                                   group = group), 
              color = "white", size = 0.1) +
    coord_equal() +
    theme_map() +
    labs(x = NULL, 
         y = NULL, 
         title = "Switzerland's regional demographics", 
         subtitle = "Average age in Swiss municipalities, 2015", 
         caption = "Geometries: ThemaKart, BFS; Data: BFS, 2016") +
  # now the discrete-option is used, 
  # and we use guide_legend instead of guide_colourbar
  scale_fill_viridis(
    option = "magma",
    name = "Average age",
    discrete = T,
    direction = -1,
    guide = guide_legend(
     keyheight = unit(5, units = "mm"),
     title.position = 'top',
     reverse = T
  ))
p
```

<img src="/wp-content/uploads/2016/12/tm-basic-map-viridis-horizontal-quantile-1.png" width="100%" />

Wow! Now that is some regional variability ;-). But there is still a
huge caveat: In my opinion, quantile scales are optimal at showing
intra-dataset-variability, but sometimes this variability can be
exaggerated. Most of the municipalities here are in the region between
39 and 43 years. The second caveat is that the legend looks somehow ugly
with all these decimals, and that people are probably having problems
interpreting such differently sized classes. That's why I am trying
"pretty breaks" in the next step, and this is basically also what you
see in almost all choropleths used for (data-)journalistic purposes.

### Discrete classes with pretty breaks {#discrete-classes-with-pretty-breaks}

``` r
# here I define equally spaced pretty breaks - 
# they will be surrounded by the minimum value at 
# the beginning and the maximum value at the end. 
# One could also use something like c(39,39.5,41,42.5,43), 
# this totally depends on the data and your personal taste.
pretty_breaks <- c(39,40,41,42,43)
# find the extremes
minVal <- min(map_data$avg_age_15, na.rm = T)
maxVal <- max(map_data$avg_age_15, na.rm = T)
# compute labels
labels <- c()
brks <- c(minVal, pretty_breaks, maxVal)
# round the labels (actually, only the extremes)
for(idx in 1:length(brks)){
  labels <- c(labels,round(brks[idx + 1], 2))
}

labels <- labels[1:length(labels)-1]
# define a new variable on the data set just as above
map_data$brks <- cut(map_data$avg_age_15, 
                     breaks = brks, 
                     include.lowest = TRUE, 
                     labels = labels)

brks_scale <- levels(map_data$brks)
labels_scale <- rev(brks_scale)

p <- ggplot() +
    # municipality polygons
    geom_polygon(data = map_data, aes(fill = brks, 
                                      x = long, 
                                      y = lat, 
                                      group = group)) +
    # municipality outline
    geom_path(data = map_data, aes(x = long, 
                                   y = lat, 
                                   group = group), 
              color = "white", size = 0.1) +
    coord_equal() +
    theme_map() +
    theme(legend.position = "bottom") +
    labs(x = NULL, 
         y = NULL, 
         title = "Switzerland's regional demographics", 
         subtitle = "Average age in Swiss municipalities, 2015", 
         caption = "Geometries: ThemaKart, BFS; Data: BFS, 2016")
q <- p +
    # now we have to use a manual scale, 
    # because only ever one number should be shown per label
    scale_fill_manual(
          # in manual scales, one has to define colors, well, manually
          # I can directly access them using viridis' magma-function
          values = rev(magma(6)),
          breaks = rev(brks_scale),
          name = "Average age",
          drop = FALSE,
          labels = labels_scale,
          guide = guide_legend(
            direction = "horizontal",
            keyheight = unit(2, units = "mm"),
            keywidth = unit(70 / length(labels), units = "mm"),
            title.position = 'top',
            # I shift the labels around, the should be placed 
            # exactly at the right end of each legend key
            title.hjust = 0.5,
            label.hjust = 1,
            nrow = 1,
            byrow = T,
            # also the guide needs to be reversed
            reverse = T,
            label.position = "bottom"
          )
      )

q
```

<img src="/wp-content/uploads/2016/12/tm-discrete-classes-pretty-breaks-1.png" width="100%" />

Now we have classes with the ranges 33.06 to 39, 39 to 40, 40 to 41, and
so on... So four classes are of the same size and the two classes with
the extremes are differently sized. One option to communicate this is to
make their respective legend keys wider than usual. `ggplot2` doesn't
have a standard option for that, so I had to dig deep into the
underlying `grid` package and extract the relevant `grobs` and change
their widths. All of the following numbers are the result of trying and
trying around. I have not yet fully understood how that system actually
works and certainly, it could be made more versatile. Something for next
christmas...

### More intuitive legend {#more-intuitive-legend}

``` r
extendLegendWithExtremes <- function(p){
  p_grob <- ggplotGrob(p)
  legend <- gtable_filter(p_grob, "guide-box")
  legend_grobs <- legend$grobs[[1]]$grobs[[1]]
  # grab the first key of legend
  legend_first_key <- gtable_filter(legend_grobs, "key-3-1-1")
  legend_first_key$widths <- unit(2, units = "cm")
  # modify its width and x properties to make it longer
  legend_first_key$grobs[[1]]$width <- unit(2, units = "cm")
  legend_first_key$grobs[[1]]$x <- unit(0.15, units = "cm")

  # last key of legend
  legend_last_key <- gtable_filter(legend_grobs, "key-3-6-1")
  legend_last_key$widths <- unit(2, units = "cm")
  # analogous
  legend_last_key$grobs[[1]]$width <- unit(2, units = "cm")
  legend_last_key$grobs[[1]]$x <- unit(1.02, units = "cm")

  # grab the last label so we can also shift its position
  legend_last_label <- gtable_filter(legend_grobs, "label-5-6")
  legend_last_label$grobs[[1]]$x <- unit(2, units = "cm")
  
  # Insert new color legend back into the combined legend
  legend_grobs$grobs[legend_grobs$layout$name == "key-3-1-1"][[1]] <- 
    legend_first_key$grobs[[1]]
  legend_grobs$grobs[legend_grobs$layout$name == "key-3-6-1"][[1]] <- 
    legend_last_key$grobs[[1]]
  legend_grobs$grobs[legend_grobs$layout$name == "label-5-6"][[1]] <- 
    legend_last_label$grobs[[1]]
  
  # finally, I need to create a new label for the minimum value 
  new_first_label <- legend_last_label$grobs[[1]]
  new_first_label$label <- round(min(map_data$avg_age_15, na.rm = T), 2)
  new_first_label$x <- unit(-0.15, units = "cm")
  new_first_label$hjust <- 1
  
  legend_grobs <- gtable_add_grob(legend_grobs, 
                                  new_first_label, 
                                  t = 6, 
                                  l = 2, 
                                  name = "label-5-0", 
                                  clip = "off")
  legend$grobs[[1]]$grobs[1][[1]] <- legend_grobs
  p_grob$grobs[p_grob$layout$name == "guide-box"][[1]] <- legend
  
  # the plot is now drawn using this grid function
  grid.newpage()
  grid.draw(p_grob)
}
extendLegendWithExtremes(q)
```

<img src="/wp-content/uploads/2016/12/tm-discrete-classes-better-legend-1.png" width="100%" />

### Better colors for classes {#better-colors-for-classes}

Almost perfect. What I still don't like is the very bright yellow color
of the first class. It makes it difficult to see the borders of the
municipalities with that color. Also I find the color of the last class
a bit too dark. That's why I now use the `magma` function with 8 classes
and strip of the first and last class.

``` r
p <- p + scale_fill_manual(
  # magma with 8 classes
  values = rev(magma(8)[2:7]),
  breaks = rev(brks_scale),
  name = "Average age",
  drop = FALSE,
  labels = labels_scale,
  guide = guide_legend(
    direction = "horizontal",
    keyheight = unit(2, units = "mm"),
    keywidth = unit(70/length(labels), units = "mm"),
    title.position = 'top',
    title.hjust = 0.5,
    label.hjust = 1,
    nrow = 1,
    byrow = T,
    reverse = T,
    label.position = "bottom"
  )
)
# reapply the legend modification from above
extendLegendWithExtremes(p)
```

<img src="/wp-content/uploads/2016/12/tm-discrete-classes-better-colors-1.png" width="100%" />

A beauty!

Relief {#relief}
------

What's needed now to give it a boost of aesthetic value is the relief of
the Swiss Alps. Every mountain lover will appreciate that.

I add the relief with `geom_raster`. Now the problem is that I can't use
the `fill` aesthetic because it (or its scale) is already in use by the
`geom_polygon` layer. The workaround is using the `alpha` aesthetic
which works fine here because the relief should be displayed with a
greyscale anyway.

``` r
p <- ggplot() +
    # raster comes as the first layer, municipalities on top
    geom_raster(data = relief, aes(x = x, 
                                  y = y, 
                                  alpha = value)) +
    # use the "alpha hack"
    scale_alpha(name = "", range = c(0.6, 0), guide = F)  + 
    # municipality polygons
    geom_polygon(data = map_data, aes(fill = brks, 
                                      x = long, 
                                      y = lat, 
                                      group = group)) +
    # municipality outline
    geom_path(data = map_data, aes(x = long, 
                                   y = lat, 
                                   group = group), 
              color = "white", size = 0.1) +
    # apart from that, nothing changes
    coord_equal() +
    theme_map() +
    theme(legend.position = "bottom") +
    labs(x = NULL, 
         y = NULL, 
         title = "Switzerland's regional demographics", 
         subtitle = "Average age in Swiss municipalities, 2015", 
         caption = "Geometries: ThemaKart, BFS; Data: BFS, 2016; Relief: swisstopo, 2016") + 
    scale_fill_manual(
      values = rev(magma(8)[2:7]),
      breaks = rev(brks_scale),
      name = "Average age",
      drop = FALSE,
      labels = labels_scale,
      guide = guide_legend(
        direction = "horizontal",
        keyheight = unit(2, units = "mm"), 
        keywidth = unit(70/length(labels), units = "mm"),
        title.position = 'top',
        title.hjust = 0.5,
        label.hjust = 1,
        nrow = 1,
        byrow = T,
        reverse = T,
        label.position = "bottom"
      )
    )
extendLegendWithExtremes(p)
```

<img src="/wp-content/uploads/2016/12/tm-with-relief-1.png" width="100%" />

Final map {#final-map}
---------

What follows are a couple of adjustments concerning:

-   font colors
-   the position of the title
-   the plot margins, i.e.: how to make better use of the available
    space and show the map as big as possible
-   smaller and less prominent caption at the bottom

Most of that happens in the additional `theme` specifications. Again,
this is just tediously trying out values after values after values...

To my great joy I also discovered that there is an `alpha` argument to
the `magma` function, which gives the colors a certain pastel tone and
make the map look even more geo-hipsterish (if you ask me).

``` r
p <- ggplot() +
    # municipality polygons
    geom_raster(data = relief, aes_string(x = "x", 
                                          y = "y", 
                                          alpha = "value")) +
    scale_alpha(name = "", range = c(0.6, 0), guide = F)  + 
    geom_polygon(data = map_data, aes(fill = brks, 
                                      x = long, 
                                      y = lat, 
                                      group = group)) +
    # municipality outline
    geom_path(data = map_data, aes(x = long, 
                                   y = lat, 
                                   group = group), 
              color = "white", size = 0.1) +
    coord_equal() +
    theme_map() +
    theme(
      legend.position = c(0.5, 0.03),
      legend.text.align = 0,
      legend.background = element_rect(fill = alpha('white', 0.0)),
      legend.text = element_text(size = 7, hjust = 0, color = "#4e4d47"),
      plot.title = element_text(hjust = 0.5, color = "#4e4d47"),
      plot.subtitle = element_text(hjust = 0.5, color = "#4e4d47", 
                                   margin = margin(b = -0.1, 
                                                   t = -0.1, 
                                                   l = 2, 
                                                   unit = "cm"), 
                                   debug = F),
      legend.title = element_text(size = 8),
      plot.margin = unit(c(.5,.5,.2,.5), "cm"),
      panel.spacing = unit(c(-.1,0.2,.2,0.2), "cm"),
      panel.border = element_blank(),
      plot.caption = element_text(size = 6, 
                                  hjust = 0.92, 
                                  margin = margin(t = 0.2, 
                                                  b = 0, 
                                                  unit = "cm"), 
                                  color = "#939184")
    ) +
    labs(x = NULL, 
         y = NULL, 
         title = "Switzerland's regional demographics", 
         subtitle = "Average age in Swiss municipalities, 2015", 
         caption = "Map CC-BY-SA; Author: Timo Grossenbacher (@grssnbchr), Geometries: ThemaKart, BFS; Data: BFS, 2016; Relief: swisstopo, 2016") + 
    scale_fill_manual(
      values = rev(magma(8, alpha = 0.8)[2:7]),
      breaks = rev(brks_scale),
      name = "Average age",
      drop = FALSE,
      labels = labels_scale,
      guide = guide_legend(
        direction = "horizontal",
        keyheight = unit(2, units = "mm"),
        keywidth = unit(70/length(labels), units = "mm"),
        title.position = 'top',
        title.hjust = 0.5,
        label.hjust = 1,
        nrow = 1,
        byrow = T,
        reverse = T,
        label.position = "bottom"
      )
    )
extendLegendWithExtremes(p)
```

<img src="/wp-content/uploads/2016/12/tm-final-map-1.png" width="100%" />

Thanks for reading, I hope you learned something. Producing high-quality
graphics like these with pure `ggplot2` is sometimes more an art than a
science and veeeeeeeryyyyy tedious, and it would probably be way easier
to export the map at an early stage and make adjustments in Illustrator
or another vector editor. But then, I just like the thought of a fully
automagic, reproducible workflow, it's almost an obsession. The big
challenge here is to put everything into a more versatile function, or
even a package, that can produce maps like these with arbitrary scales
(discrete, continuous, quantiles, pretty breaks, whatever) and arbitrary
geo data (for the US, for example).

If you think this example can be improved in any way, please use the
comment function below. I'd also be very happy to see this map adapted
to other geographic regions and/or other datasets.

As always: Follow me on [Twitter](https://twitter.com/grssnbchr)!

Update, January 2nd, 2017 {#update-january-2nd-2017}
-------------------------

This blog post has gone quite through the roof. For example, it was
featured on the [Revolution Analytics
blog](http://blog.revolutionanalytics.com/2016/12/swiss-map.html). One
guy even [printed the map and hung it on the
wall](https://twitter.com/JerryVermanen/status/814087499773526016)!

I have also received a lot of constructive feedback in the meantime. I
especially appreciated the discussions on the [RStats
Subreddit](https://www.reddit.com/r/rstats/comments/5kirj0/this_highly_aesthetic_choropleth_map_was_made/),
particularly the one about the legend / color scale.

Based on that discussion I decided to make a slightly altered version of
the color scale so one can compare the visual effect.

``` r
# same code as above but different breaks
pretty_breaks <- c(40,42,44,46,48)
# find the extremes
minVal <- min(map_data$avg_age_15, na.rm = T)
maxVal <- max(map_data$avg_age_15, na.rm = T)
# compute labels
labels <- c()
brks <- c(minVal, pretty_breaks, maxVal)
# round the labels (actually, only the extremes)
for(idx in 1:length(brks)){
  labels <- c(labels,round(brks[idx + 1], 2))
}

labels <- labels[1:length(labels)-1]
# define a new variable on the data set just as above
map_data$brks <- cut(map_data$avg_age_15, 
                     breaks = brks, 
                     include.lowest = TRUE, 
                     labels = labels)

brks_scale <- levels(map_data$brks)
labels_scale <- rev(brks_scale)

p <- ggplot() +
    # municipality polygons
    geom_raster(data = relief, aes_string(x = "x", 
                                          y = "y", 
                                          alpha = "value")) +
    scale_alpha(name = "", range = c(0.6, 0), guide = F)  + 
    geom_polygon(data = map_data, aes(fill = brks, 
                                      x = long, 
                                      y = lat, 
                                      group = group)) +
    # municipality outline
    geom_path(data = map_data, aes(x = long, 
                                   y = lat, 
                                   group = group), 
              color = "white", size = 0.1) +
    coord_equal() +
    theme_map() +
    theme(
      legend.position = c(0.5, 0.03),
      legend.text.align = 0,
      legend.background = element_rect(fill = alpha('white', 0.0)),
      legend.text = element_text(size = 7, hjust = 0, color = "#4e4d47"),
      plot.title = element_text(hjust = 0.5, color = "#4e4d47"),
      plot.subtitle = element_text(hjust = 0.5, color = "#4e4d47", 
                                   margin = margin(b = -0.1, 
                                                   t = -0.1, 
                                                   l = 2, 
                                                   unit = "cm"), 
                                   debug = F),
      legend.title = element_text(size = 8),
      plot.margin = unit(c(.5,.5,.2,.5), "cm"),
      panel.spacing = unit(c(-.1,0.2,.2,0.2), "cm"),
      panel.border = element_blank(),
      plot.caption = element_text(size = 6, 
                                  hjust = 0.92, 
                                  margin = margin(t = 0.2, 
                                                  b = 0, 
                                                  unit = "cm"), 
                                  color = "#939184")
    ) +
    labs(x = NULL, 
         y = NULL, 
         title = "Switzerland's regional demographics", 
         subtitle = "Average age in Swiss municipalities, 2015", 
         caption = "Map CC-BY-SA; Author: Timo Grossenbacher (@grssnbchr), Geometries: ThemaKart, BFS; Data: BFS, 2016; Relief: swisstopo, 2016") + 
    scale_fill_manual(
      values = rev(magma(8, alpha = 0.8)[2:7]),
      breaks = rev(brks_scale),
      name = "Average age",
      drop = FALSE,
      labels = labels_scale,
      guide = guide_legend(
        direction = "horizontal",
        keyheight = unit(2, units = "mm"),
        keywidth = unit(70/length(labels), units = "mm"),
        title.position = 'top',
        title.hjust = 0.5,
        label.hjust = 1,
        nrow = 1,
        byrow = T,
        reverse = T,
        label.position = "bottom"
      )
    )
extendLegendWithExtremes(p)
```

<img src="/wp-content/uploads/2016/12/tm-final-map-different-scale-1.png" width="100%" />

Notice that I extended the range of the first class from 33.06-39 to
33.06-40 and that, now, the classes in the "middle" have a range of two
years rather than one year. This has the advantage that both "extreme"
classes' ranges are now a bit more similar, but of course, the first is
still a lot smaller than the last. I would say the disadvantage of this
approach is that now some "visual balance" between both extremes is
lost, mostly due to the fact that a lot of municipalities have an
average age below 40 years. However, it has the other advantage that 
the really "old" municipalities at the far-right of the scale can now 
be more easily identified.


At this point, it might make sense to look at the histogram of the
municipalities:

``` r
ggplot(data = data, aes(x = avg_age_15)) + 
  geom_histogram(binwidth = 0.5) +
  theme_minimal() +
  xlab("Average age in Swiss municipality, 2015") +
  ylab("Count")
```

<img src="/wp-content/uploads/2016/12/tm-histogram-1.png" width="100%" />

As you can see, the municipalities are almost normally distributed, with
most municipalities being in the range between 39 and 43 years (&gt;75%,
look at the quantiles computation below). From that perspective, the
first class configuration might still be "closer" to the data.

``` r
quantile(data$avg_age_15)
```

    ##       0%      25%      50%      75%     100% 
    ## 33.05566 39.99845 41.65980 43.37581 66.61538

But what do I know.

No, really: This is a very difficult problem. The choice of a certain
color scale greatly alters the visual perception of the underlying
spatial patterns. I remember from my Geography studies that there are
guidelines on how to handle this (anyone got a good link, by the way?),
but there is no wrong or right. It'd be nice if you posted your opinion
about that in the comments!

One last note: Some people seem to have had problems with the `maptools`
package. In case you're wondering, here is the setup I used to run the
script in the first place:

    R version 3.3.1 (2016-06-21)
    Platform: x86_64-pc-linux-gnu (64-bit)
    Running under: Ubuntu 16.04.1 LTS

    locale:
     [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C               LC_TIME=de_CH.UTF-8        LC_COLLATE=en_US.UTF-8    
     [5] LC_MONETARY=de_CH.UTF-8    LC_MESSAGES=en_US.UTF-8    LC_PAPER=de_CH.UTF-8       LC_NAME=C                 
     [9] LC_ADDRESS=C               LC_TELEPHONE=C             LC_MEASUREMENT=de_CH.UTF-8 LC_IDENTIFICATION=C       

    attached base packages:
    [1] grid      stats     graphics  grDevices utils     datasets  methods   base     

    other attached packages:
    [1] gtable_0.2.0  dplyr_0.5.0   viridis_0.3.4 ggplot2_2.2.0 raster_2.5-8  rgdal_1.1-10  sp_1.2-3      rgeos_0.3-21 

    loaded via a namespace (and not attached):
     [1] Rcpp_0.12.7      knitr_1.14       magrittr_1.5     maptools_0.8-40  munsell_0.4.3    colorspace_1.2-6 lattice_0.20-34 
     [8] R6_2.1.3         plyr_1.8.4       tools_3.3.1      DBI_0.5-1        digest_0.6.10    lazyeval_0.2.0   assertthat_0.1  
    [15] tibble_1.2       gridExtra_2.2.1  formatR_1.4      labeling_0.3     scales_0.4.1     foreign_0.8-66
