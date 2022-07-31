---
title: "Mapa test"
author: "Edgar Caballero"
date: "2022-07-29"
output: 
  html_document:
            keep_md: yes
            toc: true
            toc_depth: 5
            toc_float:
              collapsed: false
              smooth_scroll: true
---



## Cargar paquetes y librerias


```r
require(pacman)
```

```
## Loading required package: pacman
```

```r
pacman::p_load(raster, rgdal, rgeos, stringr, sf, tidyverse, RColorBrewer, cowplot, ggpubr, 
               ggspatial, rnaturalearth, rnaturalearthdata)
rm(list = ls())
```

## cargar los datos




```r
getwd()
```

```
## [1] "C:/Users/edgar/OneDrive/Documentos/R/alchisme_analysis"
```

```r
shp <- shapefile('./shp/cacao_prd.shp')
vls <- read_csv('data_crop.csv') %>% 
  mutate(NOMBRE_DPT = iconv(NOMBRE_DPT, to = 'latin1'),
         NOMBRE_DPT = str_to_sentence(NOMBRE_DPT))
```

```
## Rows: 34 Columns: 2
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: ","
## chr (1): NOMBRE_DPT
## dbl (1): crp
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
dpt <- st_read('./shp/dptos_col.shp') %>% 
  mutate(NOMBRE_DPT = str_to_sentence(NOMBRE_DPT)) %>% 
  inner_join(., vls, by = c('NOMBRE_DPT' = 'NOMBRE_DPT'))
```

```
## Reading layer `dptos_col' from data source 
##   `C:\Users\edgar\OneDrive\Documentos\R\alchisme_analysis\shp\dptos_col.shp' 
##   using driver `ESRI Shapefile'
## Simple feature collection with 34 features and 4 fields
## Geometry type: MULTIPOLYGON
## Dimension:     XY
## Bounding box:  xmin: -81.73882 ymin: -4.228614 xmax: -66.84722 ymax: 13.39736
## Geodetic CRS:  WGS 84
```

```r
crd <- as_tibble(cbind(as.data.frame(st_coordinates(st_centroid(dpt))), dpto = dpt$NOMBRE_DPT)) 
```

```
## Warning in st_centroid.sf(dpt): st_centroid assumes attributes are constant over
## geometries of x
```

```r
wrl <- ne_countries(scale = "medium", returnclass = "sf")
```


```r
col_bb = st_as_sfc(st_bbox(dpt))
ext <- wrl %>% 
  filter(region_wb == 'Latin America & Caribbean') %>% 
  as(., 'Spatial') %>% 
  extent()
```


```r
dp1 <- dpt %>% filter(NOMBRE_DPT != 'Archipiélago de san andrés, providencia y santa catalina')
cr1 <- crd %>% filter(dpto != 'Archipiélago de san andrés, providencia y santa catalina')

dp2 <- dpt %>% filter(NOMBRE_DPT == 'Archipiélago de san andrés, providencia y santa catalina')
cr1 <- crd %>% filter(dpto != 'Archipiélago de san andrés, providencia y santa catalina')
```


```r
g1 <- ggplot() +
  geom_sf(data = wrl, fill = "white") +
  geom_sf(data = col_bb, fill = NA, color = 'red', size = 1.2) +
  ggtitle(label = 'Macrolocalización') +
  theme_bw() + 
  coord_sf(xlim = ext[1:2], ylim = ext[3:4]) +
  theme(panel.grid.major = element_blank(),
        panel.grid.minor = element_blank(),
        axis.text.x = element_blank(),
        axis.text.y = element_blank(),
        plot.title = element_text(hjust = 0.5, size = 7, face = "bold")) 

g2 <- ggplot() +
  geom_sf(data = wrl, fill = 'white') +
  geom_sf(data = dp1, aes(fill = crp)) +
  ggtitle(label = 'Área sembrada de Cacao por departamento en Colombia') +
  scale_fill_gradientn(name = 'Área sembrada (ha)', 
                       colours = RColorBrewer::brewer.pal(n = 8, name = 'YlOrBr'), 
                       na.value = 'white') +
  # scale_fill_viridis_c(option = "plasma", trans = "sqrt") +
  annotation_scale(location = "br", width_hint = 0.5) +
  annotation_north_arrow(location = "br", which_north = "true", 
                           pad_x = unit(0.1, "in"), pad_y = unit(0.2, "in"), # 0.2 # 0.3
                         style = north_arrow_fancy_orienteering) +
  coord_sf(xlim = extent(dpt)[1:2], ylim = extent(dpt)[3:4]) +
  geom_text(data= cr1, aes(x = X, y = Y, label = dpto),
            color = "darkblue", fontface = "bold", check_overlap = FALSE, size = 3.3) +
  theme_bw() +
  theme(plot.title = element_text(hjust = 0.5, size = 18, face = "bold"),
        panel.grid.major = element_blank(),
        # legend.key.width = unit(5, 'line'),
        panel.grid.minor = element_blank(),
        legend.justification = c(0,0),
        legend.position = c(0.005, 0.005),
        legend.key.size = unit(0.9, "cm"),
        legend.background = element_rect(fill = alpha('white', 1), colour = alpha('white', 0.4))) +
  labs(x = 'Longitud',
       y = 'Latitud',
       caption = "Fuente: Min. de Agricultura\n y desarrollor rural (2017)") +
  annotate(geom = 'text', x = -68, y = -3, label = 'Autor: Fabio A. \nCastro', 
           fontface = 'italic', color = 'grey22', size = 4) 

g3 <- ggplot() +
  geom_sf(data = dp2, fill = "white") +
  ggtitle(label = 'San Andres y Providencia') +
  theme_bw() + 
  theme(panel.grid.major = element_blank(),
        panel.grid.minor = element_blank(),
        axis.text.x = element_blank(),
        axis.text.y = element_blank(),
        legend.position = 'none',
        plot.title = element_text(hjust = 0.5, size = 7, face = "bold")) 

gg_inset <- ggdraw() +
  draw_plot(g2) +
  draw_plot(g1, x = 0.72, y = 0.76, width = 0.28, height = 0.19) +
  draw_plot(g3, x = 0.03, y = 0.77, width = 0.25, height = 0.17)

ggsave(plot = gg_inset,
       filename = './myMap_inset.png', units = 'in', width = 8, height = 10, dpi = 300)
```

