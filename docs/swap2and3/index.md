# Swap2and3 Search Test Analysis
<a href = 'https://meta.wikimedia.org/wiki/User:CXie_(WMF)'>Chelsy Xie</a> (Analysis & Report)  
`r as.character(Sys.Date(), '%d %B %Y')`  



## Executive Summary

Wikimedia Engineering’s Discovery’s Search team ran an A/B test from April 7 to April 25, 2016 to see how much our users care about the position of the result vs the actual content of the result by swapping the second and third search results. We found that position 2 has a higher clickthrough rate than position 3 in both group, but the clickthrough rate of third results in test group is higher than that in control group. We also found that test group users are less likely to click on the second result first than the control group, while they are more likely to click on the third result first. Additionally, test group users seems to be more satisfied with the result after they click since they are more likely to stay longer and scroll on the visited page. Therefore, we guess that users tend to look at only the first two results, when they find the first two are not relevant, some of them start to look at the third result and care more about the actual content.

## Data

```r
data1 <- data1 %>% filter(
  (event_action == 'searchResultPage' & !is.na(event_hitsReturned)) |
  (event_action == 'click' & !is.na(event_position) & event_position > -1) |
  (event_action == 'visitPage' & !is.na(event_pageViewId)) |
  (event_action == 'checkin' & !is.na(event_checkin) & !is.na(event_pageViewId))
)
data1$event_subTest[is.na(data1$event_subTest)] <- "Control"
data1 <- data1[!duplicated(data1$id), ]
data1$date <- lubridate::ymd(substr(data1$timestamp, 1,8))
data1 <- data1 %>%
  group_by(date, event_subTest, event_mwSessionId, event_searchSessionId) %>%
  filter("searchResultPage" %in% event_action) %>%
  ungroup %>% as.data.frame() # remove search_id without SERP asscociated
## 6474 out of 301603 search sessions fall into both control and test group... Delete those sessions
temp <- data1 %>% group_by(event_searchSessionId) %>% summarise(count=length(unique(event_subTest)))
data1 <- data1[data1$event_searchSessionId %in% temp$event_searchSessionId[temp$count==1], ]; rm(temp)
```

We ran this test from April 7 - April 25, 2016. Only full text search are affected by this test. Around half of the traffic were put into the test group randomly, where we swapped their second and third search results. The rest of the traffic were seen as control group. We collected a total of 1.2M events from 295.1K unique sessions.

```r
data_summary <- data1 %>%
  group_by(`Test group` = event_subTest) %>%
  summarize(`Search sessions` = length(unique(event_searchSessionId)), `Events recorded` = n()) %>% ungroup %>%
  {
    rbind(., tibble(
      `Test group` = "Total",
      `Search sessions` = sum(.$`Search sessions`),
      `Events recorded` = sum(.$`Events recorded`)
    ))
  } %>%
  mutate(`Search sessions` = prettyNum(`Search sessions`, big.mark = ","),
         `Events recorded` = prettyNum(`Events recorded`, big.mark = ","))
knitr::kable(data_summary, format = "markdown", align = c("l", "l", "r", "r"))
```



|Test group |Search sessions | Events recorded|
|:----------|:---------------|---------------:|
|Control    |157,713         |         675,211|
|swap2and3  |137,416         |         483,407|
|Total      |295,129         |       1,158,618|

As in [other A/B test analysis](https://wikimedia-research.github.io/Discovery-Search-Test-BM25/#serp_de-duplication) we did before, there is an issue with the event logging that when the user goes to the next page of search results or clicks the Back button after visiting a search result, a new page ID is generated for the search results page. The page ID is how we connect click events to search result page events. For this analysis, we de-duplicated by connecting search engine results page (searchResultPage) events that have the exact same search query, and then connected click events together based on the searchResultPage connectivity.


```r
# De-duplication
temp <- data1 %>%
  filter(event_action == "searchResultPage") %>%
  group_by(event_mwSessionId, event_searchSessionId, event_query) %>%
  mutate(new_page_id = min(event_pageViewId)) %>%
  ungroup %>%
  select(c(event_pageViewId, new_page_id)) %>%
  distinct
data1 <- left_join(data1, temp, by = "event_pageViewId"); rm(temp)
data1$new_page_id[is.na(data1$new_page_id)] <- data1$event_pageViewId[is.na(data1$new_page_id)] 
temp <- data1 %>%
  filter(event_action == "searchResultPage") %>%
  arrange(new_page_id, timestamp) %>%
  mutate(dupe = duplicated(new_page_id, fromLast = FALSE)) %>%
  select(c(id, dupe))
data1 <- left_join(data1, temp, by = "id"); rm(temp)
data1$dupe[data1$event_action != "searchResultPage"] <- FALSE
data1 <- data1[!data1$dupe & !is.na(data1$new_page_id), ] %>%
  select(-c(event_pageViewId, dupe)) %>%
  rename(page_id = new_page_id) %>%
  arrange(date, event_mwSessionId, event_searchSessionId, page_id, desc(event_action), timestamp)
# Summarize on a page-by-page basis for each SERP:
searches <- data1 %>%
  group_by(`test group` = event_subTest, event_mwSessionId, event_searchSessionId, page_id) %>%
  filter("searchResultPage" %in% event_action) %>% # keep only searchResultPage and click
  summarize(timestamp = timestamp[1], 
            results = ifelse(event_hitsReturned[1] > 0, "some", "zero"),
            clickthrough = "click" %in% event_action,
            `no. results clicked` = length(unique(event_position))-1,
            `first clicked result's position` = ifelse(clickthrough, na.omit(event_position)[1][1], NA), #There are some search with click, but position is NA
            `Clicked on position 2` = 2 %in% event_position,
            `Clicked on position 3` = 3 %in% event_position
            ) %>%
  arrange(timestamp)
```
After de-duplication, we collapsed 1M events into 483.9K searches.

```r
# Summary Table
events_summary2 <- data1 %>%
  group_by(`Test group` = event_subTest) %>%
  summarize(`Search sessions` = length(unique(event_searchSessionId)), `Events recorded` = n()) %>% ungroup %>%
  {
    rbind(., tibble(
      `Test group` = "Total",
      `Search sessions` = sum(.$`Search sessions`),
      `Events recorded` = sum(.$`Events recorded`)
    ))
  } %>%
  mutate(`Search sessions` = prettyNum(`Search sessions`, big.mark = ","),
         `Events recorded` = prettyNum(`Events recorded`, big.mark = ","))
searches_summary <- searches %>%
  group_by(`Test group` = `test group`) %>%
  summarize(`Search sessions` = length(unique(event_searchSessionId)), `Searches recorded` = n()) %>% ungroup %>%
  {
    rbind(., tibble(
      `Test group` = "Total",
      `Search sessions` = sum(.$`Search sessions`),
      `Searches recorded` = sum(.$`Searches recorded`)
    ))
  } %>%
  mutate(`Search sessions` = prettyNum(`Search sessions`, big.mark = ","),
         `Searches recorded` = prettyNum(`Searches recorded`, big.mark = ","))
knitr::kable(inner_join(searches_summary, events_summary2, by=c("Test group", "Search sessions")), 
             format = "markdown", align = c("l", "r", "r", "r")) 
```



|Test group | Search sessions| Searches recorded| Events recorded|
|:----------|---------------:|-----------------:|---------------:|
|Control    |         157,713|           279,749|         599,925|
|swap2and3  |         137,416|           204,167|         438,227|
|Total      |         295,129|           483,916|       1,038,152|


```r
# Summarize on a page-by-page basis for each visitPage:
clickedResults <- data1 %>%
  group_by(test_group = event_subTest, event_mwSessionId, event_searchSessionId, page_id) %>%
  filter("visitPage" %in% event_action) %>% #only checkin and visitPage action
  summarize(timestamp = timestamp[1], 
            position = na.omit(event_position)[1][1],
            dwell_time=ifelse("checkin" %in% event_action, max(event_checkin, na.rm=T), 0),
            scroll=sum(event_scroll)>0) %>%
  arrange(timestamp)
clickedResults$dwell_time[is.na(clickedResults$dwell_time)] <- 0
# 74148 clickedResults
clickedResults$status <- ifelse(clickedResults$dwell_time=="420", 1, 2)
```

Last but not least, it is worth noting that there are some issues in our data collecting process:

* Sometimes click events were not recorded while visitPage events were. This problem were solved in June 2016 by [T137262](https://phabricator.wikimedia.org/T137262). For this analysis, we treat both click event and visitPage event as a "click" when computing clickthrough rate, i.e. if there is either a click or a visitPage event in a session, we will say there is a clickthrough in this session.
* There were 6474 out of 301603 search sessions falling into both control and test buckets. We deleted those sessions in data cleansing step.

## Results

### Engagement
Firstly, we compare the overall clickthrough rate between control and test group. We've checked that there is no significant difference in zero result rate between these two groups. The plot below shows that for each session, the control group has a significantly higher engagement; for each SERP, the test group has higher engagement, but the difference is not significant.

```r
# Compare overall CTR
# per session:
engagement_overall_session <- data1 %>% 
  group_by(event_subTest, event_searchSessionId) %>%
  summarise(clickthrough = "click" %in% event_action|"visitPage" %in% event_action, results = sum(event_hitsReturned, na.rm=TRUE) > 0) %>%
  filter(results == TRUE) %>%
  group_by(event_subTest) %>%
  summarise(clicks = sum(clickthrough), session = n()) %>%
  cbind(binom:::binom.bayes(.$clicks, n=.$session)[, c("mean", "lower", "upper")]) 
  #knitr::kable(engagement_overall_session, format = "markdown", align = c("l", "r", "r", "r", "r", "r"))

# per SERP:
engagement_overall_serp <- searches %>%
  filter(results == "some") %>%
  group_by(`test group`) %>%
  summarise(clicks = sum(clickthrough), searches = n()) %>%
  cbind(binom:::binom.bayes(.$clicks, n=.$searches, tol=.Machine$double.eps^0.6)[, c("mean", "lower", "upper")])
  #knitr::kable(engagement_overall_serp, format = "markdown", align = c("l", "r", "r", "r", "r", "r"))

# plot
p_ctr_ses <- engagement_overall_session %>%
  ggplot(aes(x = 1, y = mean, color = event_subTest)) +
  geom_pointrange(aes(ymin = lower, ymax = upper), position = position_dodge(width = 1)) +
  scale_color_brewer("Test Group", palette = "Set1", guide = guide_legend(ncol = 2)) +
  scale_y_continuous(labels = scales::percent_format(), expand = c(0.01, 0.01)) +
  labs(x = NULL, y = "Clickthrough rate",
       title = "Proportion of sessions that end in \n at least one click/visitPage event") +
  geom_text(aes(label = sprintf("%.1f%%", 100 * mean), y = upper + 0.0025, vjust = "bottom"),
            position = position_dodge(width = 1)) +
  theme(legend.position = "bottom")
p_ctr_serp <- engagement_overall_serp %>%
  ggplot(aes(x = 1, y = mean, color = `test group`)) +
  geom_pointrange(aes(ymin = lower, ymax = upper), position = position_dodge(width = 1)) +
  scale_color_brewer("Test Group", palette = "Set1", guide = guide_legend(ncol = 2)) +
  scale_y_continuous(labels = scales::percent_format(), expand = c(0.01, 0.01)) +
  labs(x = NULL, y = "Clickthrough rate",
       title = "Proportion of SERPs that end in \n at least one click event") +
  geom_text(aes(label = sprintf("%.1f%%", 100 * mean), y = upper + 0.0025, vjust = "bottom"),
            position = position_dodge(width = 1)) +
  theme(legend.position = "bottom")
plot_grid(p_ctr_ses, p_ctr_serp)
```

![](index_files/figure-html/engagement_overall-1.png)<!-- -->

Next, we compare the clickthrough rates on second and third position by test groups. All the differences in the graph below are significant. In both control and test group, the clickthrough rates on position 2 are higher than position 3. When comparing the same position between groups, the engagement of 3rd results in test group is higher than that in control group, but it is not as high as its counterpart, the 2nd results in control group. 


```r
# Compare position 2 and 3
# per session, among all nonzero result session:
engagement_2and3_session <- data1 %>% 
  group_by(event_subTest, event_searchSessionId) %>%
  summarise(clickthrough2 = 2 %in% event_position, clickthrough3 = 3 %in% event_position, 
            clickthrough = "click" %in% event_action|"visitPage" %in% event_action, results = sum(event_hitsReturned, na.rm=TRUE) > 0) %>%
  filter(results == TRUE) %>%
  group_by(event_subTest) %>%
  summarise(clicked2 = sum(clickthrough2), clicked3 = sum(clickthrough3), session = n()) %>%
  cbind(binom:::binom.bayes(.$clicked2, n=.$session)[, c("mean", "lower", "upper")]) %>%
  rename(ctr2 = mean, lower2 = lower, upper2 = upper) %>%
  cbind(binom:::binom.bayes(.$clicked3, n=.$session)[, c("mean", "lower", "upper")]) %>%
  rename(ctr3 = mean, lower3 = lower, upper3 = upper)
# per SERP, among all nonzero result session:
engagement_2and3_serp <- searches %>%
  filter(results == "some") %>%
  group_by(`test group`) %>%
  summarise(clicked2 = sum(`Clicked on position 2`), clicked3 = sum(`Clicked on position 3`), searches = n()) %>%
  cbind(binom:::binom.bayes(.$clicked2, n=.$searches, tol=.Machine$double.eps^0.7)[, c("mean", "lower", "upper")]) %>%
  rename(ctr2 = mean, lower2 = lower, upper2 = upper) %>%
  cbind(binom:::binom.bayes(.$clicked3, n=.$searches, tol=.Machine$double.eps^0.7)[, c("mean", "lower", "upper")]) %>%
  rename(ctr3 = mean, lower3 = lower, upper3 = upper)

# plot
p_ctr23_ses <- engagement_2and3_session[, c("ctr2", "lower2", "upper2")] %>%
  rbind(setNames(engagement_2and3_session[, c("ctr3", "lower3", "upper3")], names(.))) %>%
  mutate(`test group` = c("Control_position2", "swap2and3_position2", "Control_position3", "swap2and3_position3")) %>%
  ggplot(aes(x = 1, y = ctr2, color = `test group`)) +
  geom_pointrange(aes(ymin = lower2, ymax = upper2), position = position_dodge(width = 1)) +
  scale_color_brewer("Test Group", palette = "Set1", guide = guide_legend(ncol = 2)) +
  scale_y_continuous(labels = scales::percent_format(), expand = c(0.01, 0.01)) +
  labs(x = NULL, y = "Clickthrough rate",
       title = "Proportion of sessions that end in \n at least one click/visitPage event on position x") +
  geom_text(aes(label = sprintf("%.1f%%", 100 * ctr2), y = upper2 + 0.0025, vjust = "bottom"),
            position = position_dodge(width = 1)) +
  theme(legend.position = "bottom")
p_ctr23_serp <- engagement_2and3_serp[, c("ctr2", "lower2", "upper2")] %>%
  rbind(setNames(engagement_2and3_serp[, c("ctr3", "lower3", "upper3")], names(.))) %>%
  mutate(`test group` = c("Control_position2", "swap2and3_position2", "Control_position3", "swap2and3_position3")) %>%
  ggplot(aes(x = 1, y = ctr2, color = `test group`)) +
  geom_pointrange(aes(ymin = lower2, ymax = upper2), position = position_dodge(width = 1)) +
  scale_color_brewer("Test Group", palette = "Set1", guide = guide_legend(ncol = 2)) +
  scale_y_continuous(labels = scales::percent_format(), expand = c(0.01, 0.01)) +
  labs(x = NULL, y = "Clickthrough rate",
       title = "Proportion of SERPs that end in \n at least one click event on position x") +
  geom_text(aes(label = sprintf("%.1f%%", 100 * ctr2), y = upper2 + 0.0025, vjust = "bottom"),
            position = position_dodge(width = 1)) +
  theme(legend.position = "bottom")
plot_grid(p_ctr23_ses, p_ctr23_serp)
```

![](index_files/figure-html/engagement_2and3-1.png)<!-- -->

### First Clicked Result’s Position

We can see that test group users are less likely to click on the second result first than the control group, while they are more likely to click on the third result first. There is no significant difference in other positions. 


```r
safe_ordinals <- function(x) {
  return(vapply(x, toOrdinal::toOrdinal, ""))
}
first_clicked <- searches %>%
  filter(results == "some" & clickthrough & !is.na(`first clicked result's position`)) %>%
  mutate(`first clicked result's position` = ifelse(`first clicked result's position` < 5, safe_ordinals(`first clicked result's position`), "5th or higher")) %>%
  group_by(`test group`, `first clicked result's position`) %>%
  tally %>%
  mutate(total = sum(n), prop = n/total) %>%
  ungroup
temp <- as.data.frame(binom:::binom.bayes(first_clicked$n, n = first_clicked$total)[, c("mean", "lower", "upper")])
first_clicked <- cbind(first_clicked, temp); rm(temp)
first_clicked %>%
  ggplot(aes(x = 1, y = mean, color = `test group`)) +
  geom_pointrange(aes(ymin = lower, ymax = upper), position = position_dodge(width = 1)) +
  geom_text(aes(label = sprintf("%.1f", 100 * prop), y = upper + 0.0025, vjust = "bottom"),
            position = position_dodge(width = 1)) +
  scale_y_continuous(labels = scales::percent_format(),
                     expand = c(0, 0.005), breaks = seq(0, 1, 0.01)) +
  scale_color_brewer("Test Group", palette = "Set1", guide = guide_legend(ncol = 2)) +
  facet_wrap(~ `first clicked result's position`, scale = "free_y", nrow = 1) +
  labs(x = NULL, y = "Proportion of searches",
       title = "Position of the first clicked result",
       subtitle = "With 95% credible intervals") +
  theme(legend.position = "bottom",
        panel.grid.major.x = element_blank(),
        panel.grid.minor.x = element_blank(),
        axis.ticks.x = element_blank(),
        axis.text.x = element_blank(),
        strip.background = element_rect(fill = "gray90"),
        panel.border = element_rect(color = "gray30", fill = NA))
```

![](index_files/figure-html/first_clicked-1.png)<!-- -->

### Dwell Time per Visited Page

First we compare the overall survival curves between the two groups. Surprisingly, the test group users are significantly more likely to stay longer on visited pages. 


```r
# Compare overall dwell time 
temp <- clickedResults
temp$SurvObj <- with(temp, survival::Surv(dwell_time, status == 2))
fit_all <- survival::survfit(SurvObj ~ test_group, data = temp)
survminer::ggsurvplot(fit_all, conf.int = TRUE, xlab="T (Dwell Time in seconds)", ylab="Proportion of visits longer than T (P%)", 
                      surv.scale = "percent", palette="Set1", legend="bottom", legend.title = "Test Group", legend.labs=c("Control","swap2and3"))
```

![](index_files/figure-html/dwelltime_overall-1.png)<!-- -->

```r
rm(temp)
```

Next we compare the survival curves for users who clicked on second or third result. We can see that users in test group who clicked on 3rd result have longer dwell time than others. Users in test group who clicked on 2nd result are more likely to leave the visited page at the beginning, and those who stay tends to stay longer.


```r
temp <- clickedResults %>% filter(position %in% c(2,3)) %>%
  mutate(Test_Group = paste(test_group, paste0("position",position), sep="_"))
temp$SurvObj <- with(temp, survival::Surv(dwell_time, status == 2))
fit_2and3 <- survival::survfit(SurvObj ~ Test_Group, data = temp)
survminer::ggsurvplot(fit_2and3, conf.int = TRUE, xlab="T (Dwell Time in seconds)", ylab="Proportion of visits longer than T (P%)", 
                      surv.scale = "percent", palette="Set1", legend="bottom", legend.title = "Test Group",
                      legend.labs=c("Control_position2","Control_position3","swap2and3_position2","swap2and3_position3"))
```

![](index_files/figure-html/dwelltime_2and3-1.png)<!-- -->

```r
rm(temp)
```

### Scroll

Users in the test group are more likely to scroll on the visited page.


```r
# Compare overall scroll proportion
scroll_overall <- clickedResults %>%
  group_by(test_group) %>%
  summarize(scrolls=sum(scroll), visits=n(), proportion = sum(scroll)/n()) %>%
  ungroup
scroll_overall <- cbind(
  scroll_overall,
  as.data.frame(
    binom:::binom.bayes(
      scroll_overall$scrolls,
      n = scroll_overall$visits)[, c("mean", "lower", "upper")]
  )
)
scroll_overall %>%
  ggplot(aes(x = 1, y = mean, color = test_group)) +
  geom_pointrange(aes(ymin = lower, ymax = upper), position = position_dodge(width = 1)) +
  scale_color_brewer("Test Group", palette = "Set1", guide = guide_legend(ncol = 2)) +
  scale_y_continuous(labels = scales::percent_format(), expand = c(0.01, 0.01)) +
  labs(x = NULL, y = "Proportion of visits",
       title = "Proportion of visits with scroll by test group") +
  geom_text(aes(label = sprintf("%.1f%%", 100 * proportion), y = upper + 0.0025, vjust = "bottom"),
            position = position_dodge(width = 1)) +
  theme(legend.position = "bottom")
```

![](index_files/figure-html/scroll_overall-1.png)<!-- -->

Users in test group who clicked on the 3rd result are more likely to scroll on the visited pages, but all the differences are not statistically significant.


```r
# Compare position 2 and 3
scroll_2and3 <- clickedResults %>%
  filter(position %in% c(2,3)) %>%
  mutate(`Test Group` = paste(test_group, paste0("position",position), sep="_")) %>%
  group_by(`Test Group`) %>%
  summarize(scrolls=sum(scroll), visits=n(), proportion = sum(scroll)/n()) %>%
  ungroup
scroll_2and3 <- cbind(
  scroll_2and3,
  as.data.frame(
    binom:::binom.bayes(
      scroll_2and3$scrolls,
      n = scroll_2and3$visits)[, c("mean", "lower", "upper")]
  )
)
scroll_2and3 %>%
  ggplot(aes(x = 1, y = mean, color = `Test Group`)) +
  geom_pointrange(aes(ymin = lower, ymax = upper), position = position_dodge(width = 1)) +
  scale_color_brewer("Test Group", palette = "Set1", guide = guide_legend(ncol = 2)) +
  scale_y_continuous(labels = scales::percent_format(), expand = c(0.01, 0.01)) +
  labs(x = NULL, y = "Proportion of visits",
       title = "Proportion of visits with scroll by test group") +
  geom_text(aes(label = sprintf("%.1f%%", 100 * proportion), y = upper + 0.0025, vjust = "bottom"),
            position = position_dodge(width = 1)) +
  theme(legend.position = "bottom")
```

![](index_files/figure-html/scroll_2and3-1.png)<!-- -->

## Discussion

Since we only swap the second and third result in this test and the results above are somewhat confusing, it is hard to conclude that users care more about position than actual content of the result. Instead, we guess that users tend to look at only the first two results, when they find the first two are not relevant, some of them start to look at the third result and care more about the actual content. This explains why the overall engagement is lower in test group (since the second results in test group are less relevant), and why the clickthrough rates on position 2 are always higher than position 3, but engagement of 3rd results in test group is higher than that in control group. It also explains why test group users are more likely to click on the third result first, why test group users who click on the 3rd result have a longer dwell time, and why test group has a longer dwell time overall and more likely to scroll on the visited page (because test group users who are confused by the swapping process but still open the result care more about the actual content). However, without further experiment, we cannot prove our guess.

Additionally, instead of making explicit control buckets, we simply treat those users who didn't get assigned to test group as control group users. We suspect that this behavior results in putting all users who don't have the new javascript into control group automatically, so some metrics may be biased.
