---
layout: post
title: The 7 Roles of Basketball
subtitle: A study of Basketball Positions using Cluster Analysis
---

This is the documentation of the study on the true positions in today’s
basketball, using cluster analysis. The full article detailing the
results is [here](https://medium.com/p/2cf6ab757667/edit).

### Necessary Packages

These are the packages that need to be loaded for the analysis:

``` r
library(nbastatR)
library(tidyverse)
library(RColorBrewer)
library(ggrepel)
library(broom)
library(patchwork)
library(gghighlight)
library(colorspace)
```

All packages can be installed using the `install.packages()` function,
except for nbastatR, in which case
`devtools::install_github("abresler/nbastatR")` should be used instead.

### Data import and manipulation

First, the data from the nbastatR API was collected. The active players
in the 2018-19 season were gathered on an initial data frame, along with
their general statistics and information. Then, the numeric statistics
were converted to per game measures and the position variable was
re-categorized so that each player was classified as either guard (G),
forward (F) or center (C). Finally, the players whose team variable was
missing were ruled out.

``` r
# Team Rosters
roster <- seasons_rosters(2019, return_message = F) %>% 
  select(idTeam, slugTeam, idPlayer, groupPosition) %>% 
  rename(id_team = idTeam, team = slugTeam, pos = groupPosition)

players <- seasons_players(seasons = 2019, nest_data = F, return_message = F) %>% 
  select(idPlayer, namePlayer) 

general_stats <- teams_players_stats(seasons = 2019, 
                             types = 'player', 
                             tables = 'general', 
                             season_types = 'Regular Season', 
                             measures = 'Base', 
                             modes = 'Totals', 
                             defenses = 'Overall', 
                             assign_to_environment = F, 
                             return_message = F) %>% 
  unnest(cols = dataTable) %>% 
  select(idPlayer, agePlayer, slugTeam, idTeam,
         minutes, gp, wins, losses,
         fgm, fga, pctFG, 
         fg3m, fg3a, pctFG3,
         ftm, fta, pctFT,
         pts, ast, oreb, dreb, treb, blk, blka, stl, pf, tov)

data <- players %>% 
  left_join(general_stats, by = 'idPlayer')

data2 <- data %>% 
  left_join(roster, by = 'idPlayer') %>% 
  mutate(slugTeam = ifelse(is.na(slugTeam), team, slugTeam),
         idTeam = ifelse(is.na(idTeam), id_team, idTeam)) %>% 
  select(idPlayer, namePlayer, pos, everything(), -team, -id_team) %>% 
  mutate(pos = case_when(pos == 'G-F' ~ 'G',
                         pos == 'F-G' ~ 'F',
                         pos == 'C-F' ~ 'C',
                         pos == 'F-C' ~ 'F',
                         TRUE ~ pos)) %>% 
  mutate_at(vars(7, 11, 12, 14, 15, 17, 18, 20:29), function(x) round(x/data$gp, digits = 2)) %>% 
  filter(!is.na(slugTeam)) %>% 
  mutate_if(is.numeric, gtools::na.replace, replace = 0) %>% 
  mutate_at(vars(minutes, pctFG3), round, digits = 3)
```

After this, the dataset was fed with the data that was going to be used
on the clustering:

``` r
cluster_data1 <- data2

get_freq_data <- function(df, category){
  data <- synergy(seasons = 2019, result_types = c("player"),
                  season_types = c("Regular Season"),
                  set_types = c("offensive"),
                  categories = c("Transition", 
                                 'Isolation', 
                                 'PRBallHandler', 
                                 "PRRollman", 
                                 'Postup',
                                 'SpotUp',
                                 'Handoff', 
                                 'Cut', 
                                 'OffScreen', 
                                 'OffRebound'),
                  assign_to_environment = F, 
                  nest_data = T,
                  return_message = F)
  varname <- ifelse(str_detect(category, '^Off.'), 
                    str_to_lower(str_c(str_extract(category, '^[A-Z]'), str_extract(category, '(?<=Off).'), '_freq')),
                    str_c(str_to_lower(str_sub(category, 1, 3)), '_freq'))
  res <- df %>% 
    left_join(data[which(data$categorySynergy == category),]$dataSynergy %>% 
                data.frame %>%  
                select(idPlayer, pctFrequency) %>% 
                rename(!!varname := pctFrequency), 
              by = 'idPlayer')
  return(res)
}

cluster_data2 <- cluster_data1 %>% 
  get_freq_data('Transition') %>% 
  get_freq_data('Isolation') %>% 
  get_freq_data('PRBallHandler') %>% 
  get_freq_data('PRRollman') %>% 
  get_freq_data('Postup') %>% 
  get_freq_data('Handoff') %>% 
  get_freq_data('SpotUp') %>% 
  get_freq_data('Cut') %>% 
  get_freq_data('OffScreen') %>% 
  get_freq_data('OffRebound')

cluster_data_final <- cluster_data2 %>%
  mutate_if(is.numeric,  gtools::na.replace, replace = 0) %>% 
  mutate(sum = tra_freq + iso_freq + prb_freq + prr_freq + 
           pos_freq + spo_freq + han_freq + cut_freq + os_freq + or_freq) %>% 
  filter(sum > 0) %>% 
  select(-sum)

sample_n(cluster_data_final, 10)
```

    ## # A tibble: 10 x 39
    ##    idPlayer namePlayer pos   agePlayer slugTeam idTeam minutes    gp  wins
    ##       <dbl> <chr>      <chr>     <dbl> <chr>     <dbl>   <dbl> <dbl> <dbl>
    ##  1   203487 Michael C… G            27 ORL      1.61e9    13.3    28    17
    ##  2   101139 CJ Miles   F            32 MEM      1.61e9    16.2    53    33
    ##  3  1628421 Devin Rob… F            24 WAS      1.61e9    13.5     7     1
    ##  4   202695 Kawhi Leo… F            28 TOR      1.61e9    34      60    41
    ##  5   203895 Jordan Mc… G            28 WAS      1.61e9    12.3    27     9
    ##  6   202687 Bismack B… C            26 CHA      1.61e9    14.5    54    24
    ##  7  1628365 Markelle … G            21 ORL      1.61e9    22.5    19    12
    ##  8   202326 DeMarcus … C            28 GSW      1.61e9    25.7    30    23
    ##  9  1628964 Mo Bamba   C            21 ORL      1.61e9    16.3    47    19
    ## 10   203584 Troy Dani… G            27 PHX      1.61e9    14.9    51    13
    ## # … with 30 more variables: losses <dbl>, fgm <dbl>, fga <dbl>, pctFG <dbl>,
    ## #   fg3m <dbl>, fg3a <dbl>, pctFG3 <dbl>, ftm <dbl>, fta <dbl>, pctFT <dbl>,
    ## #   pts <dbl>, ast <dbl>, oreb <dbl>, dreb <dbl>, treb <dbl>, blk <dbl>,
    ## #   blka <dbl>, stl <dbl>, pf <dbl>, tov <dbl>, tra_freq <dbl>, iso_freq <dbl>,
    ## #   prb_freq <dbl>, prr_freq <dbl>, pos_freq <dbl>, han_freq <dbl>,
    ## #   spo_freq <dbl>, cut_freq <dbl>, os_freq <dbl>, or_freq <dbl>

### Clusters

Before modelling, were excluded the players that had all clustering
variables missing. With the data ready, the cluster analysis proceeded
using the K-means algotithm. After fitting several models and analysing
the elbow plot, it seemed reasonable to use 7 clusters. That being said,
a final model with said number of groups was fitted, and the results
were binded in the original dataset:

``` r
model_df <- cluster_data_final %>% filter(gp > 29) %>% select(30:39) %>% mutate_all(round, digits = 3)

res <- data.frame(VARIAB = NA, CENTERS = NA)
set.seed(111)
for (i in 1:20){
  mod <- kmeans(model_df, centers = i, nstart = 50)
  res <- base::rbind(res, c(mod$tot.withinss, i))
}
res <- res %>% na.omit

ggplot(res, aes(x = CENTERS, y = VARIAB)) +  #Elbow Plot
  geom_line() + 
  geom_point() 
```

![Elbow Plot](/img/the-7-roles-of-basketball/cluster-1.png)<!-- -->

``` r
set.seed(111)
clusters <- kmeans(model_df, centers = 7, nstart = 50)

# Bind clusters to original data frame
model_result <- augment(x = clusters, data = filter(cluster_data_final, gp > 29)) %>% 
  rename(cluster = .cluster) %>%
  mutate(cluster = case_when(cluster == 1 ~ 'Big Man Shooter',
                             cluster == 2 ~ 'Dynamic Shooter',
                             cluster == 3 ~ 'Primary Ball Handler',
                             cluster == 4 ~ 'Big Man Post Up',
                             cluster == 5 ~ 'Static Shooter',
                             cluster == 6 ~ 'Secondary Ball Handler',
                             cluster == 7 ~ 'Big Man Rim Runner'))
```

### Visualization

After fitting the model, a few graphs were plotted to visualize the
results. The clusters were called ‘roles’, as they represented the
playstyle of each athlete. This is the visualization of the clustering,
achieved using PCA to get the two principal components that most explain
the variance:

``` r
# Color scales
cluster_colors <- qualitative_hcl(n = 7) # Customized colors for each role
names(cluster_colors) <- unique(model_result$cluster)
scale_fill_cluster <- scale_fill_manual(name = "Role", values = cluster_colors) 
scale_color_cluster <- scale_color_manual(name = "Role", values = cluster_colors) 

pos_colors <- brewer.pal(3, 'Set1') # Customized Colors for each position
names(pos_colors) <- unique(na.omit(model_result$pos))
scale_fill_pos <- scale_fill_manual(name = "Position", values = pos_colors, 
                                    labels = c('G'= 'Guard', 'F' = 'Forward', 'C' = 'Center')) 
# Clusters Visualization

set.seed(111)
pca <- prcomp(model_result[,30:39], center = T, scale = T)
df_cluster_plot <- bind_cols(model_result, as_tibble(pca$x))
hl_players <- df_cluster_plot %>% 
  group_by(cluster) %>% 
  top_n(10, minutes)

cluster_graph <- ggplot(df_cluster_plot, aes(x = PC1, y = PC2, fill = cluster)) +
  stat_ellipse(geom = 'polygon', alpha = 0.40) + 
  geom_point(data = hl_players,
             mapping = aes(color = cluster), 
             show.legend = F) + 
  geom_text_repel(data = hl_players, 
                  mapping = aes(label = namePlayer, color = cluster),
                  fontface = 'bold',
                  size = 5,
                  show.legend = F) +
  scale_fill_cluster + 
  scale_color_cluster + 
  labs(title = 'The 7 Basketball Roles',
       subtitle = "The closer a player is to a group, \nbigger the resamblance between it's playstyle and the group's",
       caption = 'Highlighted, the 10 players with the most usage rates') + 
  guides(fill = guide_legend(title.theme = element_blank(), nrow = 1)) +
  theme(axis.title.x = element_blank(),
        axis.text.x = element_blank(),
        axis.title.y = element_blank(),
        axis.text.y = element_blank(),
        axis.ticks = element_blank(),
        plot.title = element_text(size = 18, face = 'bold', hjust = 0.5), 
        plot.subtitle = element_text(size = 14, hjust = 0.5),
        plot.caption = element_text(size = 14, hjust = 1),
        legend.position = 'top',
        legend.text = element_text(size = 12),
        legend.title = element_blank(), 
        panel.background = element_blank(),
        panel.grid = element_line(color = 'gray92'))
cluster_graph
```

![](/img/the-7-roles-of-basketball/viz-1.png)<!-- -->

Some graphs showing the distribution of the roles and it’s relation to
the original positions:

``` r
# Role and position distribution on the NBA
role_count <- model_result %>% 
  count(cluster, name = 'count') %>%
  mutate(prop = count/sum(count))

role_graph <- ggplot(role_count, aes(y = prop, x = reorder(cluster, prop), fill = cluster)) + 
  geom_col() +
  coord_flip() +
  scale_y_continuous(labels = scales::label_percent(accuracy = 1)) +
  labs(title = 'Role Distribution on the NBA', y = '% of Players') +
  scale_fill_cluster +
  guides(fill = F) +
  theme(axis.title.y = element_blank(),
        axis.text.y = element_text(size = 12, margin = margin(r = 5)),
        axis.ticks.y  = element_blank(),
        axis.title.x = element_text(size = 12, margin = margin(t = 12)),
        axis.text.x = element_text(size = 10),
        plot.title = element_text(size = 18, face = 'bold', hjust = 0.5),
        panel.background = element_blank(),
        panel.grid = element_line(color = 'grey92'))

position_count <- model_result %>% 
  count(pos, name = 'count') %>% 
  mutate(prop = count/sum(count)) %>% 
  na.omit

position_graph <- ggplot(position_count, aes(x = reorder(pos, prop), y = prop, fill = pos)) + # Positions
  geom_col() + 
  labs(title = 'Position Distribution on the NBA', y = '% of Players') + 
  scale_fill_pos +
  scale_x_discrete(labels = c('G'= 'Guard', 'F' = 'Forward', 'C' ='Center')) +
  scale_y_continuous(labels = scales::label_percent(1)) +
  coord_flip() +
  guides(fill = F) +
  theme(axis.title.y = element_blank(),
        axis.text.y = element_text(size = 12, margin = margin(r = 5)),
        axis.ticks.y  = element_blank(),
        axis.title.x = element_text(size = 12, margin = margin(t = 12)),
        axis.text.x = element_text(size = 10),
        plot.title = element_text(size = 18, face = 'bold', hjust = 0.5),
        panel.background = element_blank(),
        panel.grid = element_line(color = 'grey92'))

position_graph + role_graph
```

![](/img/the-7-roles-of-basketball/viz2-1.png)<!-- -->

``` r
# Distribution of Positions on each Role
df <- model_result %>% 
  na.omit() %>% 
  group_by(cluster, pos) %>% 
  tally(name = 'count') %>% 
  mutate(perc = count/sum(count)) %>% 
  ungroup %>% 
  mutate(lab = case_when(pos == 'G' ~ 'Guard', pos == 'F' ~ 'Forward', pos == 'C' ~ 'Center'))

positions_by_role <- ggplot(df, aes(x = reorder(cluster, count), y = perc, 
                                    fill = pos)) + 
  geom_col() + 
  coord_flip() +
  guides(fill = guide_legend(label.position = 'top',
                             keywidth = 5, 
                             keyheight = 1.5, 
                             reverse = T)) +
  labs(x = NULL, y = '% of Players',
       title = 'Position Distribution per Role') +
  scale_fill_pos +
  scale_y_continuous(labels = scales::percent) +
  theme(axis.text.x = element_text(size = 12),
        axis.title.x = element_text(size = 14, margin = margin(t = 10)),
        axis.title.y = element_blank(),
        axis.text.y = element_text(size = 12, margin = margin(r = 5)),
        strip.text.x = element_text(size = 14),
        legend.position = 'top',
        legend.title = element_blank(),
        legend.text = element_text(size = rel(1.2)),
        plot.title = element_text(size = 18, face = 'bold', hjust = 0.5, margin = margin(b = 10)))
positions_by_role
```

![](/img/the-7-roles-of-basketball/viz3-1.png)<!-- -->

``` r
# Roles Proportion per Team
teams_conf <- nbastatR::standings(2019, 'Regular Season', nest_data = F, return_message = F) %>% 
  select(idTeam, nameConference) %>% 
  rename(conf = nameConference) %>% 
  mutate(conf = str_c(conf, 'ern'))
```

    ## Missing QUARTER_PTS in dictionary

``` r
prop_role_team <- model_result %>% 
  group_by(idTeam, cluster) %>% 
  tally(name = 'count') %>% 
  mutate(tot = sum(count), prop = count/tot) %>% 
  ungroup() %>% 
  complete(idTeam, cluster, fill = list(count = 0, tot = 0, prop = 0)) %>% 
  left_join(teams_conf, by = 'idTeam') %>% 
  left_join(distinct(model_result, idTeam, slugTeam), by = 'idTeam') %>% 
  select(slugTeam, cluster, conf, count, tot, prop)

role_team_graph <- ggplot(data = prop_role_team, 
                          mapping = aes(x = fct_rev(slugTeam), y = prop, fill = fct_rev(cluster))) + 
  geom_col(position = 'stack', alpha = 0.85) + 
  coord_flip() +
  scale_fill_cluster +
  scale_y_continuous(labels = scales::label_percent(accuracy = 1)) +
  facet_wrap(~ conf, scales = 'free') +
  labs(title = "Players's Role Distribution per Team",
       x = 'Teams',
       y = '% of Players') +
  guides(fill = guide_legend(nrow = 1, reverse = T)) + 
  theme(axis.text.x = element_text(size = 12),
        axis.title.x = element_text(size = 14),
        axis.title.y = element_blank(),
        axis.text.y = element_text(size = 12, margin = margin(r = 5)),
        axis.ticks.y = element_blank(),
        legend.position = 'top',
        legend.title = element_blank(),
        legend.text = element_text(size = 14),
        plot.title = element_text(size = 18, face = 'bold', hjust = 0.5, margin = margin(b = 15)),
        strip.text = element_text(size = 12, face = 'bold'))
role_team_graph
```

![](/img/the-7-roles-of-basketball/viz4-1.png)<!-- -->

Finally, here are the average of points, assists and rebounds per role,
alongside the league’s average for comparison:

``` r
# Average Points, Assits and Rebounds
cluster_averages <- model_result %>% 
  select(-idPlayer, -idTeam) %>% 
  group_by(cluster) %>% 
  summarise_if(is.numeric, list( ~ mean(., na.rm = T), ~ sd(., na.rm = T))) 

points_graph <- ggplot(cluster_averages, aes(x = reorder(cluster, pts_mean), y = pts_mean, color = cluster)) + 
  geom_pointrange(aes(ymin = pts_mean - pts_sd, ymax = pts_mean + pts_sd)) +
  geom_hline(aes(yintercept = mean(model_result$pts)), color = 'gray50', lwd = 1.2, alpha = 0.5) + 
  coord_flip() +
  scale_color_cluster +
  guides(color = F) + 
  labs(x = NULL, y = 'Points',
       title = 'Average Points per Role') + 
  theme(plot.title = element_text(size = 18, face = 'bold', hjust = 0, margin = margin(b = 30)),
        axis.text.y= element_text(size = 10),
        axis.text.x= element_text(size = 10),
        axis.title.x = element_text(size = 12), 
        axis.ticks.y = element_blank(),
        panel.background = element_blank(),
        panel.grid = element_line(color = 'gray92')) +
  annotate(geom = 'text', x = 1, y = mean(model_result$pts) + 2.6, 
           label = "League's Average", 
           fontface = 'bold', 
           alpha = 0.5)

assists_graph <- ggplot(cluster_averages, aes(x = reorder(cluster, ast_mean), y = ast_mean, color = cluster)) + 
  geom_pointrange(aes(ymin = ast_mean - ast_sd, ymax = ast_mean + ast_sd)) +
  geom_hline(aes(yintercept = mean(model_result$ast)), color = 'gray50', lwd = 1.2, alpha = 0.5) + 
  coord_flip() +
  scale_color_cluster +
  guides(color = F) + 
  labs(x = NULL, y = 'Assists',
       title = 'Average Assists per Role') + 
  theme(plot.title = element_text(size = 18, face = 'bold', hjust = 0, margin = margin(b = 30)),
        axis.text.y= element_text(size = 10),
        axis.text.x= element_text(size = 10),
        axis.title.x = element_text(size = 12), 
        axis.ticks.y = element_blank(),
        panel.background = element_blank(),
        panel.grid = element_line(color = 'gray92')) +
  annotate(geom = 'text', x = 1, y = mean(model_result$ast) + 1, 
           label = "League's Average", 
           fontface = 'bold', 
           alpha = 0.5)

rebounds_graph <- ggplot(cluster_averages, aes(x = reorder(cluster, treb_mean), y = treb_mean, color = cluster)) + 
  geom_pointrange(aes(ymin = treb_mean - treb_sd, ymax = treb_mean + treb_sd)) +
  geom_hline(aes(yintercept = mean(model_result$treb)), color = 'gray50', lwd = 1.2, alpha = 0.5) + 
  coord_flip() +
  scale_color_cluster +
  guides(color = F) + 
  labs(x = NULL, y = 'Rebounds',
       title = 'Average Rebounds per Role') + 
  theme(plot.title = element_text(size = 18, face = 'bold', hjust = 0, margin = margin(b = 30)),
        axis.text.y= element_text(size = 10),
        axis.text.x= element_text(size = 10),
        axis.title.x = element_text(size = 12), 
        axis.ticks.y = element_blank(),
        panel.background = element_blank(),
        panel.grid = element_line(color = 'gray92')) +
  annotate(geom = 'text', x = 1.5, y = mean(model_result$treb) + 1.8, 
           label = "League's Average", 
           fontface = 'bold', 
           alpha = 0.5)
```

![](/img/the-7-roles-of-basketball/unnamed-chunk-1-1.png)<!-- -->

![](/img/the-7-roles-of-basketball/unnamed-chunk-2-1.png)<!-- -->

![](/img/the-7-roles-of-basketball/unnamed-chunk-3-1.png)<!-- -->