# journey2work
using R to analyze commuting patterns in ontario



``` r
library(tidyverse)
```


``` r
library(cancensus)
```

    ## Census data is currently stored temporarily.
    ## 
    ##  In order to speed up performance, reduce API quota usage, and reduce unnecessary network calls, please set up a persistent cache directory by setting options(cancensus.cache_path = '<path to cancensus cache directory>')
    ## 
    ##  You may add this option, together with your API key, to your .Rprofile.

Grabbing the [commuting table](https://www12.statcan.gc.ca/census-recensement/2016/dp-pd/dt-td/Rp-eng.cfm?LANG=E&APATH=3&DETAIL=0&DIM=0&FL=A&FREE=0&GC=0&GID=0&GK=0&GRP=1&PID=113344&PRID=10&PTYPE=109445&S=0&SHOWALL=0&SUB=0&Temporal=2017&THEME=125&VID=0&VNAMEE=&VNAMEF=) from statscan.

``` r
raw_commute <- read_csv("~/projects/R stuff/commute/98-400-X2016391_English_CSV_data.csv") %>% 
    janitor::clean_names() %>% 
    select(code = geo_code_por,
           live = geo_name,
           work = geo_name_1,
           total = dim_sex_3_member_id_1_total_sex)
```


``` r
# filter only those whose code starts w/ 35
ontario_commute <- raw_commute %>% 
    filter(str_detect(code, pattern = "^35")) 
```

let's get working age pop

``` r
library(cancensus)
library(sf)
```

    ## Linking to GEOS 3.7.0, GDAL 2.3.2, PROJ 5.2.0

``` r
# create list of ontario census divisions to pass to cancensus
regions_list_ontario <- list_census_regions("CA16") %>% 
  filter(str_detect(region, pattern = "^35"))  %>% 
  as_census_region_list
```

    ## Querying CensusMapper API for regions data...

``` r
pop_data <- get_census("CA16",
                           regions = regions_list_ontario,
                           vectors = "v_CA16_61",
                           level = "CD",
                       geo_format = "sf", labels = "short") %>% 
  janitor::clean_names()
```



``` r
# clean
pop_data %>% 
  mutate(working_age = v_ca16_61) %>% 
  mutate(code = as.double(geo_uid)) %>% 
  select(code, working_age, shape_area, geometry) -> pop_data_clean
```

``` r
# remove commuting within cd, compute totals
ontario_commute %>% 
  filter(live != work) %>%
  group_by(live) %>% 
  mutate(total_commuters = sum(total),
         prop_commuters = total / total_commuters ) %>% 
  ungroup() %>% 
  left_join(pop_data_clean, by = "code") %>% 
  group_by(work) %>% 
  mutate(total_commuters_destination = sum(total)) %>% 
  mutate(live_prop = total_commuters / working_age,
         work_prop = total / working_age ) %>% 
  ungroup() -> ontario_commute_clean
```

vis time!

``` r
ontario_commute_clean %>% 
    filter(total >= 3000) %>% 
    mutate(live = str_wrap(live, width = 15)) %>% 
    mutate(live = fct_reorder(live, total)) %>% 
    mutate(work = fct_reorder(work, total)) %>% 
    ggplot(aes(live, work, fill = total)) +
    geom_tile(alpha = 0.7) +
    theme_light() +
    theme(axis.text.x = element_text(angle = 90, hjust = 1)) +
    scale_fill_viridis_c(labels = scales::comma_format()) +
    labs(fill = "# of commuters",
         title = "The largest number of commuters in Ontario are those who live in York, Peel and Durham who commute to Toronto for work",
         subtitle = "Number of employed labour force aged 15+ who commute by Census Division - Minimum 3,000 commuters to be represented",
         x = "Live - Census Division",
         y = "Work - Census Division",
         caption = "Data from Statistics Canada 2016 Canadian Census - Journey to Work \n https://www12.statcan.gc.ca/census-recensement/2016/rt-td/jtw-ddt-eng.cfm")
```

![](commute_files/figure-markdown_github/unnamed-chunk-6-1.png)

``` r
ggsave("~/projects/R stuff/commute/map.png", width = 16, height = 9)
```

let's make a map

``` r
ontario_commute_clean %>% 
  filter((!live %in% c("Rainy River", "Kenora", "Thunder Bay", "Algoma", "Nipissing", 
                       "Cochrane", "Greater Sudbury / Grand Sudbury", "Manitoulin",
                       "Timiskaming", "Sudbury", "Parry Sound"))) %>% 
  ggplot() +
  geom_sf(aes(fill = live_prop)) +
  scale_fill_viridis_c("% Labour force who commute to work", labels = scales::percent) + theme_minimal() +
  theme(panel.grid = element_blank(),
        axis.text = element_blank(),
        axis.ticks = element_blank()) + 
  coord_sf(datum=NA) +
  labs(title = "The % of the labour force (aged 15+) who commute by Census Division in Southern Ontario")
```

![](commute_files/figure-markdown_github/unnamed-chunk-8-1.png)

