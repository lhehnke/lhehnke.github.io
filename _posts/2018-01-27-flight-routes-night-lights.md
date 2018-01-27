---
layout: post
title: "Flights at Night: Mapping airline routes on NASA's night lights images"
excerpt_separator: "<!--more-->"
date: "January 27, 2018"
type: post
published: false
status: publish
categories:
- maps
tags:
- rstats
- openflights
- airlines
- flight routes
- nasa
- night lights
- maps
---

<p style="margin-top:60px">Admittedly, mapping airline routes using the well-known data set from <a href="https://openflights.org/data.html">OpenFlights.org</a> is neither a bold move nor that innovative. <a href="https://flowingdata.com/2011/05/11/how-to-map-connections-with-great-circles/">FlowingData</a>
did it, <a href="https://www.r-bloggers.com/how-to-draw-connecting-routes-on-map-with-r-and-great-circles/">R bloggers</a> did it, and <a href="http://spatial.ly/2013/05/great-world-flight-paths-map/">Spatial.ly</a> wrote about someone who did it.</p>  

Yet, as an aviation enthusiast, I couldn't resist laying my hands on this kind of data, especially after stumbling across 
<a href="https://weiminwang.blog/2015/06/24/use-r-to-plot-flight-routes-on-a-fancy-world-background/">Weimin Wang's call for action</a> for further visualizations featuring some sort of lighting effect. 
While Weimin used urban data to achieve the effect in his maps, I will draw on night lights images provided by <a href="https://earthobservatory.nasa.gov/Features/NightLights/page3.php">NASA's Earth Observatory</a>.

<!--more-->

In addition, despite me being the *n*th person to approach the general topic of working with *OpenFlights* data, most of the previous analyses don't seem to start from scratch, i.e., by downloading the raw `.dat` files 
straigth from their website and processing it prior to mapping. The code provided below guides you through both these initial steps and demonstrates how to create nice looking maps like this one

<p align="center"><img src="https://raw.githubusercontent.com/lhehnke/lhehnke.github.io/master/img/openflights-nasa/Airlines.png" width="655px" height="327x" vspace="40px"/></p>

using *OpenFlights* data in combination with *NASA* images. 

Given the bulk of tutorials on flight data, the blog post itself contains rather technical instructions. For more detailed explanations on the prerequisites and foundations of spatial mapping using `R` see the aforementioned links.

## Setup

In order to visualize *OpenFlights* airline routes, you will need the packages `data.table`, `geosphere`, `ggplot2`, `grid`, `maps`, `jpeg`, `plyr`, and `tidyverse`.

As usual, I used `p_load()` from the `pacman` package to load the required libraries at once.

```r
# Install and load pacman if not already installed
if (!require("pacman")) install.packages("pacman")
library(pacman)

# Load packages
p_load(data.table, geosphere, ggplot2, grid, jpeg, plyr, tidyverse)
```

## Downloading and importing data

To download and import the required data, simply run the following code in `R`:

```r
# Download OpenFlights data
download.file("https://raw.githubusercontent.com/jpatokal/openflights/master/data/airlines.dat",
destfile = "airlines.dat", mode = "wb")
download.file("https://raw.githubusercontent.com/jpatokal/openflights/master/data/airports.dat", 
destfile = "airports.dat", mode = "wb")
download.file("https://raw.githubusercontent.com/jpatokal/openflights/master/data/routes.dat", 
destfile = "routes.dat", mode = "wb")

# Import data
airlines <- fread("airlines.dat", sep = ",", skip = 1)
airports <- fread("airports.dat", sep = ",")
routes <- fread("routes.dat", sep = ",")
```

After obtaining the raw data, some cleaning needs to be done.  

```r
# Add column names
colnames(airlines) <- c("airline_id", "name", "alias", "iata", "icao", "callsign", "country", "active")
colnames(airports) <- c("airport_id", "name", "city", "country", "iata", "icao", "latitude", "longitude", 
"altitude", "timezone", "dst", "tz_database_time_zone", "type", "source")
colnames(routes) <- c("airline", "airline_id", "source_airport", "source_airport_id", "destination_airport", 
"destination_airport_id", "codeshare", "stops", "equipment")

# Convert character to numeric
routes$airline_id <- as.numeric(routes$airline_id)

# Join airline data with data on routes
flights <- left_join(routes, airlines, by = "airline_id")

# Join data on flights with information on airports
airports_orig <- airports[, c(5, 7, 8)]
colnames(airports_orig) <- c("source_airport", "source_airport_lat", "source_airport_long")

airports_dest <- airports[, c(5, 7, 8)]
colnames(airports_dest) <- c("destination_airport", "destination_airport_lat", "destination_airport_long")

flights <- left_join(flights, airports_orig, by = "source_airport")
flights <- left_join(flights, airports_dest, by = "destination_airport")

# Remove missing values
flights <- na.omit(flights, cols = c("source_airport_long", "source_airport_lat", "destination_airport_long", "destination_airport_lat"))
```

## Spatial data wrangling

EXPLANATION

```r
# Split the data into separate data sets
flights_split <- split(flights, flights$name)

# Calculate intermediate points between each two locations
flights_all <- lapply(flights_split, function(x) gcIntermediate(x[, c("source_airport_long", "source_airport_lat")], 
x[, c("destination_airport_long", "destination_airport_lat")], 
100, breakAtDateLine = FALSE, addStartEnd = TRUE, sp = TRUE))

# Turn data into a data frame for mapping with ggplot2
flights_fortified <- lapply(flights_all, function(x) ldply(x@lines, fortify))

# Unsplit lists
flights_fortified <- do.call("rbind", flights_fortified)

# Add and clean column with airline names
flights_fortified$name <- rownames(flights_fortified)
flights_fortified$name <- gsub("\\..*", "", flights_fortified$name)

# Subset multiple airlines
## Note: Change this code snippet if you're more interested in mapping other airlines.
flights_subset <- c("Lufthansa", "Emirates", "British Airways")
flights_subset <- flights_fortified[flights_fortified$name %in% flights_subset, ]

# Extract first and last observations for plotting source and destination points (i.e., airports)
flights_points <- flights_subset %>%
group_by(group) %>%
filter(row_number() == 1 | row_number() == n())
```

EXPLANATION ON GREAT CIRCLE DISTANCES & CHOICE OF AIRLINES (domination of US carriers in real-life but not in maps)

## Processing NASA's night lights image

EXPLANATION Satellite images of the earth at night -- the so-called "night lights" -- .... read `.jpg` file with `readJPEG()function from `jpeg` package:

```r
# Download NASA Night Lights image
download.file("https://www.nasa.gov/specials/blackmarble/2016/globalmaps/BlackMarble_2016_01deg.jpg", 
destfile = "BlackMarble_2016_01deg.jpg", mode = "wb")

# Load picture and render
earth <- readJPEG("BlackMarble_2016_01deg.jpg", native = TRUE)
earth <- rasterGrob(earth, interpolate = TRUE)
```
ADD Rendering in the second step...

## Mapping airline routes on night lights

For the upcoming graphs I slightly went overboard and added the not necessarily required and purely aesthetically motivated step of
mimicking the airlines' logos and chose color according to companies' color palette and downloaded free fonts which come rather close to the original ones:
* Lufthansa: `Helvetica black`(font); `#f9ba00` (color)
* Emirates: `Fontin`; `#ff0000`
* British Airways: `#075aaa`; `Baker Signet Std`

In `R`, adding customized fonts can be done via
```r
#install.packages("extrafont")
#library(extrafont)
#font_import(paths = "/Users/Lisa/Desktop/fonts", prompt = F)
```

### (1) Lufthansa

EXLANATION

```r
ggplot() +
annotation_custom(earth, xmin = -180, xmax = 180, ymin = -90, ymax = 90) +
geom_path(aes(long, lat, group = id, color = name), alpha = 0.0, size = 0.0, data = flights_subset) + 
geom_path(aes(long, lat, group = id, color = name), alpha = 0.2, size = 0.3, color = "#f9ba00", data = flights_subset[flights_subset$name == "Lufthansa", ]) + 
geom_point(data = flights_points[flights_points$name == "Lufthansa", ], aes(long, lat), alpha = 0.8, size = 0.1, colour = "white") +
theme(panel.background = element_rect(fill = "#05050f", colour = "#05050f"), 
panel.grid.major = element_blank(),
panel.grid.minor = element_blank(), 
axis.title = element_blank(), 
axis.text = element_blank(), 
axis.ticks.length = unit(0, "cm"),
legend.position = "none") +
annotate("text", x = -150, y = -18, hjust = 0, size = 14,
label = paste("Lufthansa"), color = "#f9ba00", family = "Helvetica Black") +
annotate("text", x = -150, y = -26, hjust = 0, size = 8, 
label = paste("Flight routes"), color = "white") +
annotate("text", x = -150, y = -30, hjust = 0, size = 7, 
label = paste("lhehnke.github.io || OpenFlights.org"), color = "white", alpha = 0.5) +
coord_equal() 
```

<p align="center"><img src="https://raw.githubusercontent.com/lhehnke/lhehnke.github.io/master/img/openflights-nasa/Lufthansa.png" width="655px" height="327x" vspace="40px"/></p>


### (2) Emirates

```r
ggplot() +
annotation_custom(earth, xmin = -180, xmax = 180, ymin = -90, ymax = 90) +
geom_path(aes(long, lat, group = id, color = name), alpha = 0.0, size = 0.0, data = flights_subset) + 
geom_path(aes(long, lat, group = id, color = name), alpha = 0.2, size = 0.3, color = "#ff0000", data = flights_subset[flights_subset$name == "Emirates", ]) + 
geom_point(data = flights_points[flights_points$name == "Emirates", ], aes(long, lat), alpha = 0.8, size = 0.1, colour = "white") +
theme(panel.background = element_rect(fill = "#05050f", colour = "#05050f"), 
panel.grid.major = element_blank(),
panel.grid.minor = element_blank(), 
axis.title = element_blank(), 
axis.text = element_blank(), 
axis.ticks.length = unit(0, "cm"),
legend.position = "none") +
annotate("text", x = -150, y = -18, hjust = 0, size = 14,
label = paste("Emirates"), color = "#ff0000", family = "Fontin") +
annotate("text", x = -150, y = -26, hjust = 0, size = 8, 
label = paste("Flight routes"), color = "white") +
annotate("text", x = -150, y = -30, hjust = 0, size = 7, 
label = paste("lhehnke.github.io || OpenFlights.org"), color = "white", alpha = 0.5) +
coord_equal() 
```

<p align="center"><img src="https://raw.githubusercontent.com/lhehnke/lhehnke.github.io/master/img/openflights-nasa/Emirates.png" width="655px" height="327x" vspace="40px"/></p>

### (3) British Airways

```r
ggplot() +
annotation_custom(earth, xmin = -180, xmax = 180, ymin = -90, ymax = 90) +
geom_path(aes(long, lat, group = id, color = name), alpha = 0.0, size = 0.0, data = flights_subset) + 
geom_path(aes(long, lat, group = id, color = name), alpha = 0.2, size = 0.3, color = "#075aaa", data = flights_subset[flights_subset$name == "British Airways", ]) + 
geom_point(data = flights_points[flights_points$name == "British Airways", ], aes(long, lat), alpha = 0.8, size = 0.1, colour = "white") +
theme(panel.background = element_rect(fill = "#05050f", colour = "#05050f"), 
panel.grid.major = element_blank(),
panel.grid.minor = element_blank(), 
axis.title = element_blank(), 
axis.text = element_blank(), 
axis.ticks.length = unit(0, "cm"),
legend.position = "none") +
annotate("text", x = -150, y = -18, hjust = 0, size = 14,
label = paste("BRITISH AIRWAYS"), color = "#075aaa", family = "Baker Signet Std") +
annotate("text", x = -150, y = -26, hjust = 0, size = 8, 
label = paste("Flight routes"), color = "white") +
annotate("text", x = -150, y = -30, hjust = 0, size = 7, 
label = paste("lhehnke.github.io || OpenFlights.org"), color = "white", alpha = 0.5) +
coord_equal() 
```

<p align="center"><img src="https://raw.githubusercontent.com/lhehnke/lhehnke.github.io/master/img/openflights-nasa/British_Airways_new.png" width="655px" height="327x" vspace="40px"/></p>

## Mapping multiple airlines' flight routes

```r
ggplot() +
annotation_custom(earth, xmin = -180, xmax = 180, ymin = -90, ymax = 90) +
geom_path(aes(long, lat, group = id, color = name), alpha = 0.2, size = 0.3, data = flights_subset) + 
geom_point(data = flights_points, aes(long, lat), alpha = 0.8, size = 0.1, colour = "white") +
scale_color_manual(values = c("#f9ba00", "#ff0000", "#075aaa")) +
theme(panel.background = element_rect(fill = "#05050f", colour = "#05050f"), 
panel.grid.major = element_blank(),
panel.grid.minor = element_blank(), 
axis.title = element_blank(), 
axis.text = element_blank(), 
axis.ticks.length = unit(0, "cm"),
legend.position = "none") +
annotate("text", x = -150, y = -4, hjust = 0, size = 14, 
label = paste("Lufthansa"), color = "#f9ba00", family = "Helvetica Black") +
annotate("text", x = -150, y = -11, hjust = 0, size = 14, 
label = paste("Emirates"), color = "#ff0000", family = "Fontin") +
annotate("text", x = -150, y = -18, hjust = 0, size = 14, 
label = paste("BRITISH AIRWAYS"), color = "#075aaa", family = "Baker Signet Std") + 
annotate("text", x = -150, y = -30, hjust = 0, size = 8, 
label = paste("Flight routes"), color = "white") +
annotate("text", x = -150, y = -34, hjust = 0, size = 7, 
label = paste("lhehnke.github.io || OpenFlights.org"), color = "white", alpha = 0.5) +
coord_equal() 
```

which will finally get you the map shown right in the beginning of this post: 

<p align="center"><img src="https://raw.githubusercontent.com/lhehnke/lhehnke.github.io/master/img/openflights-nasa/Airlines_final.png" width="655px" height="327x" vspace="40px"/></p>

And to end this post with the wise words of a famous man: *"Keep on travelin'! Ciao."* 
