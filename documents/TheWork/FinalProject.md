Forecasting the 2018 California Legislative Election
================
EJ Arce and Jonathan Matz
4/2/2017

### Importing election results data

We are importing three datasets and we will eventually join them. `LegResults6814` consists of every candidate who ran for a seat in the California State Legislature from 1968 to 2014. `LegResults16` includes such candidates who ran in 2016. `threeoffices` includes the voting totals for three US offices (Presidential, Gubernatorial, and US Senate) in each of the California senatorial and assembly districts, from 1966 to 2016. This dataset previously consisted of three separate datasets, which Jonathan and I cleaned in R or Excel, to create `threeoffices`.

``` r
LegResults6814 <- read.dta("../../data-raw/SLERs1967to2015_20160912b_CA.dta")

LegResults16 <- read_excel("../../data-raw/001_CA2016GenOfficecopy.xls")

threeoffices <- read_excel("../../data-raw/threeoffices662016copy.xlsx")
```

### Cleaning election results data

For both datasets, we want each observation to represent a given chamber's district's voting totals for both national and state legislative offices, each office including a Democratic or Republican candidate. It is certainly possible that some candidates for a legislative office ran unopposed or against a third-party candidate, resulting in some blank entries.

``` r
# Selecting and renaming variables of interest, and filtering the dataset to only include democratic and republican candidates. 

LegResults6814 <- LegResults6814 %>%
  dplyr::select(v05, v07, v09z, v21, v22, v23) %>%
  rename(year = v05,
         chamber = v07,
         District = v09z,
         party = v21,
         incumbency = v22,
         votes = v23) %>%
  filter(party == c(100, 200))
LegResults6814$chamber <- ifelse(LegResults6814$chamber == 8, "SEN", "HS")
LegResults6814$party <- ifelse(LegResults6814$party == 100, "Dem", "Repub")
LegResults6814 <- aggregate(x = LegResults6814["votes"], by = LegResults6814[c("year", "chamber", "District", "party", "incumbency")], sum)

# Coding dummy variables prez_party (1 if the president is a Democrat, -1 if the president is a Republican) and mid_penalty (1 if the president is a Democrat, -1 if the president is a Republican, 0 if the year includes a presidential election).

# Coding for dummy variable incumbent (1 if the seat up for election is held by a democrat, -1 if held by a Republican, 0 if the seat is open).

LegResults6814 <- LegResults6814  %>%
  mutate(dummyparty = ifelse(party == "Dem", 1, -1)) %>%
  mutate(incumbentparty = dummyparty * incumbency) %>%
  spread(key = party, value = votes) %>%
  filter(year > 1972)
LegResults6814[is.na(LegResults6814)] <- 0

# Repeating the same steps for LegResults16

LegResults16 <- LegResults16 %>%
  dplyr::select(v05, v07, v09, partyoriginal, v23, incumbency) %>%
  rename(year = v05,
         chamber = v07,
         District = v09,
         party = partyoriginal,
         votes = v23) %>%
  filter(party == c("Democratic", "Republican"))
LegResults16$chamber <- ifelse(LegResults16$chamber == 8, "SEN", "HS")
LegResults16$party <- ifelse(LegResults16$party == "Democratic", "Dem", "Repub")
LegResults16 <- aggregate(x = LegResults16["votes"], by = LegResults16[c("year", "chamber", "District", "party", "incumbency")], sum)
LegResults16 <- LegResults16  %>%
  mutate(dummyparty = ifelse(party == "Dem", 1, -1)) %>%
  mutate(incumbentparty = dummyparty * incumbency) %>%
  spread(key = party, value = votes) %>%
  filter(year > 1972)
LegResults16[is.na(LegResults16)] <- 0

head(LegResults6814)
```

    ##   year chamber District incumbency dummyparty incumbentparty   Dem Repub
    ## 1 1974      HS        3          0          1              0 41972     0
    ## 2 1974      HS        4          0         -1              0     0 36820
    ## 3 1974      HS        5          0          1              0 45873     0
    ## 4 1974      HS        6          0         -1              0     0 27665
    ## 5 1974      HS        6          1          1              1 64472     0
    ## 6 1974      HS        7          0         -1              0     0 33842

``` r
head(LegResults16)
```

    ##   year chamber District incumbency dummyparty incumbentparty    Dem  Repub
    ## 1 2016      HS        1          1         -1             -1      0 197166
    ## 2 2016      HS        2          1          1              1  32781      0
    ## 3 2016      HS        3          0          1              0 109521      0
    ## 4 2016      HS        3          1         -1             -1      0 178787
    ## 5 2016      HS        4          0         -1              0      0 100170
    ## 6 2016      HS        4          0          1              0 176279      0

We have selected variables of interest and created new variables that will be useful for our model. Each observation still represents how a district voted for each candidate, so we need to aggregate the votes in the Republican and Democrat candidate columns so that each observation represents how a district voted for both parties' candidates.

``` r
# Aggregating rows by votes so that our LegResults 6814 dataset is now a district's legislative election

LegResults6814 <- aggregate(x = LegResults6814[c("Dem",
                                                 "Repub",
                                                 "incumbency",
                                                 "incumbentparty",
                                                 "dummyparty")],
                            by = LegResults6814[c("year",
                                                  "chamber",
                                                  "District")],
                            sum)

LegResults16 <- aggregate(x = LegResults16[c("Dem",
                                             "Repub",
                                             "incumbency",
                                             "incumbentparty",
                                             "dummyparty")],
                          by = LegResults16[c("year",
                                              "chamber",
                                              "District")],
                          sum)
```

Our dataset on voting totals for presidential, gubernatorial, and US Senate offices was mostly tidy, and any modifications to it were made in Excel. We now only need to select the variables of interest, remove unnecessary rows (some rows contained sidenotes for the data), and rename the district column to match with our legislative results dataset.

``` r
threeoffices <- threeoffices %>%
  filter(Dummy1 == 1,
         year > 1972) %>%
  dplyr::select(year, chamber, dist, (17:24)) %>%
  rename(District = dist)
```

Now both datasets are clean, and we will join them soon. However, first we need to add data on presidential approval ratings and percent change in real income (courtesy of Gallup and the Department of Commerce). Both datasets were collected at the national level.

### Importing and cleaning presidential approval and economy strength data

``` r
Econ_State2 <- read_excel("../../data-raw/econ-strength.xls")
PrezApprove <- read_excel("../../data-raw/Gallup_Prez_Approv_1966to2016.xlsx")

PrezApprove <- PrezApprove %>%
  dplyr::select(Year, Approve) %>%
  rename(year = Year)

# For percent change in real income, we are using Dr. Klarner's weighted percent change formula. This includes attributing the highest weight to the most recent quarter. In our case, since we will test this model days before the election, the second quarter (July through September) will have the largest weight. The reason we are doing this is because we expect that voters focus more on how the economy has changed now, and less 10 months ago. First, we need to measure percent change for each quarter. Our dataset only gives raw income values, so the first mutation creates percent changes for each quarter, and our second mutation creates our weighted annual percent change.

Econ_State <- Econ_State2 %>%
  spread(key = Quarter, value = Income) %>%
  mutate(I_IIpercent = (II-I)/I,
         II_IIIpercent = (lag(III)-lag(II))/lag(II),
         III_IVpercent = (lag(IV)-lag(III))/lag(III),
         IV_Ipercent = (I - lag(IV))/lag(IV)) %>%
  mutate(perc_change = (3*I_IIpercent +
                        4*II_IIIpercent +
                        III_IVpercent +
                        2*IV_Ipercent)/10) %>%
  rename(year = Year)
```

### Joining legislative elections data with threeoffices, and joining Econ\_State and PrezApprove to Results

``` r
# First joining the legislative results from 2016 to legislative results from 1968 through 2014.

LegResults <- full_join(LegResults16, LegResults6814, by = c("year", "chamber", "District",  "Dem", "Repub", "incumbentparty", "incumbency", "dummyparty"))

# Joining the entire legislative results to US offices results.

Results <- full_join(LegResults, threeoffices, by = c("year", "chamber", "District"))

# Joining presidential approval and percent change in real income data to the Results. Also creating another presidential approval rating variable that multiplies the rating by -1 if the president is republican.

Results <- left_join(Results, Econ_State, by = "year")
Results <- left_join(Results, PrezApprove, by = "year")

# Importing 2018 empty data and joining it to include lagged effects.

df2018 <- read_excel("../../data-raw/2018template.xlsx")

df2018 <- df2018 %>%
  mutate(prez_party = -1, mid_penalty = -1)

# Adding another incumbent variable, this time to explicitly describe which party holds the incumbency. Also adding a categorical variable Leg_winner to represent which party captured the legislative seat.

Results <- Results %>%
  rename(Leg_Dem = Dem, Leg_Repub = Repub) %>%
  mutate(incumb_party_name = ifelse(incumbentparty == 1, "Democrat", ifelse(incumbentparty == 0, "No incumbent", "Republican")),
         Leg_winner = ifelse(Leg_Repub > Leg_Dem, "Republican", "Democract"),
         perc_Dem = Leg_Dem / (Leg_Dem + Leg_Repub),
         perc_Repub = Leg_Repub / (Leg_Dem + Leg_Repub)) %>%
  arrange(year, chamber, District) %>%
  mutate(perc_prez_Dem = Prez_Dem/(Prez_Dem + Prez_Repub),
         perc_sen_Dem = US_Sen_Dem/(US_Sen_Dem + US_Sen_Repub),
         perc_sen2_Dem = US_Sen2_Dem/(US_Sen2_Dem + US_Sen2_Repub),
         perc_gub_Dem = Gub_Dem/(Gub_Dem + Gub_Repub),
         Approve = Approve/100)

Results$prez_party <- ifelse(Results$year %in% c(1968, 1978,1980,1994,1996,1998,2000,2010,2012,2014,2016), 1, -1)
Results$mid_penalty <- ifelse(Results$year %in% c(1968,1972,1976,1980,1984,1988,1992,1996,2000,2004,2008,2012, 2016), 0, ifelse(Results$year %in% c(1970,1974,1982,1986,1990,2002,2006), -1, 1))
```

### Fully clean dataset

``` r
head(Results)
```

    ##   year chamber District Leg_Dem Leg_Repub incumbency incumbentparty
    ## 1 1974      HS        1      NA        NA         NA             NA
    ## 2 1974      HS        2      NA        NA         NA             NA
    ## 3 1974      HS        3   41972         0          0              0
    ## 4 1974      HS        4       0     36820          0              0
    ## 5 1974      HS        5   45873         0          0              0
    ## 6 1974      HS        6   64472     27665          1              1
    ##   dummyparty Prez_Dem Prez_Repub Gub_Dem Gub_Repub US_Sen_Dem US_Sen_Repub
    ## 1         NA       NA         NA   49033     45113      54624        35815
    ## 2         NA       NA         NA   51300     41867      58658        30320
    ## 3          1       NA         NA   41675     48068      49590        35463
    ## 4         -1       NA         NA   43442     38605      54781        25016
    ## 5          1       NA         NA   42013     38264      50049        27862
    ## 6          0       NA         NA   49335     43671      59964        30545
    ##   US_Sen2_Dem US_Sen2_Repub      I     II    III   IV I_IIpercent
    ## 1          NA            NA 1204.4 1230.4 1266.6 1296  0.02158751
    ## 2          NA            NA 1204.4 1230.4 1266.6 1296  0.02158751
    ## 3          NA            NA 1204.4 1230.4 1266.6 1296  0.02158751
    ## 4          NA            NA 1204.4 1230.4 1266.6 1296  0.02158751
    ## 5          NA            NA 1204.4 1230.4 1266.6 1296  0.02158751
    ## 6          NA            NA 1204.4 1230.4 1266.6 1296  0.02158751
    ##   II_IIIpercent III_IVpercent IV_Ipercent perc_change Approve
    ## 1    0.02440763    0.03321739  0.01363407  0.02228786    0.54
    ## 2    0.02440763    0.03321739  0.01363407  0.02228786    0.54
    ## 3    0.02440763    0.03321739  0.01363407  0.02228786    0.54
    ## 4    0.02440763    0.03321739  0.01363407  0.02228786    0.54
    ## 5    0.02440763    0.03321739  0.01363407  0.02228786    0.54
    ## 6    0.02440763    0.03321739  0.01363407  0.02228786    0.54
    ##   incumb_party_name Leg_winner  perc_Dem perc_Repub perc_prez_Dem
    ## 1              <NA>       <NA>        NA         NA            NA
    ## 2              <NA>       <NA>        NA         NA            NA
    ## 3      No incumbent  Democract 1.0000000  0.0000000            NA
    ## 4      No incumbent Republican 0.0000000  1.0000000            NA
    ## 5      No incumbent  Democract 1.0000000  0.0000000            NA
    ## 6          Democrat  Democract 0.6997406  0.3002594            NA
    ##   perc_sen_Dem perc_sen2_Dem perc_gub_Dem prez_party mid_penalty
    ## 1    0.6039872            NA    0.5208187         -1          -1
    ## 2    0.6592416            NA    0.5506241         -1          -1
    ## 3    0.5830482            NA    0.4643816         -1          -1
    ## 4    0.6865045            NA    0.5294770         -1          -1
    ## 5    0.6423868            NA    0.5233504         -1          -1
    ## 6    0.6625197            NA    0.5304496         -1          -1

### Importing CA shapefiles

To visualize our data, we would like to include a map of California senatorial and assembly districts. The shapefiles necessary to do this were obtained from the Census Bureau.

``` r
SENshape <- readShapePoly("../../data-raw/CA_SEN_shapefile/cb_2016_06_sldu_500k.shp")
HSshape <- readShapePoly("../../data-raw/CA_HS_shapefile/cb_2016_06_sldl_500k.shp")

SENshapefort <- fortify(SENshape, region = "NAME")
HSshapefort <- fortify(HSshape, region = "NAME")
```

### Some visualizations

``` r
legs <- subset(Results, !is.na(incumb_party_name))

plot1 <- ggplot(legs, aes(y = Leg_Dem, x = Leg_Repub)) +
  geom_point(mapping = aes(col = incumb_party_name)) +
  scale_colour_manual(values = c("blue", "grey", "red")) +
  geom_abline(intercept = 0, slope = 1) +
  scale_y_continuous(name = "Vote count for the Democrat",
                     labels = comma) +
  scale_x_continuous(name = "Vote count for the Republican",
                     labels = comma) +
  labs(col = "Incumbency status") +
  ggtitle("The incumbency effect")
plot1
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-9-1.png)

``` r
plot2 <- ggplot(Results, aes(y = perc_Dem, x = year)) +
  geom_smooth(mapping = aes(col = incumb_party_name)) +
  scale_colour_manual(values = c("blue", "grey", "red")) +
  geom_abline(intercept = .5, slope = 0) +
  ggtitle("The incumbency effect over time") +
  xlab("Election") +
  ylab("Proportion of votes won by the Democrat") +
  labs(col = "Incumbency status")
plot2
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-9-2.png)

``` r
# Plot 2 shows that the incumbency advantage has diminished in the past two
# decades. Thus, in our model, we should vary the incumbency variable by time.
```

``` r
# Creating a subset of Results that only includes assembly district voting totals in 2014.

HS10 <- Results %>%
  filter(year == 2010, chamber == "HS")

HS10map <- ggplot() +
  geom_map(data = HS10,
           aes(map_id = District, fill = perc_Dem),
           map = HSshapefort) +
  expand_limits(x = HSshapefort$long, y = HSshapefort$lat) +
  scale_fill_gradient2(low = muted("red"),
                       mid = "white", midpoint = .5,
                       high = muted("blue"),
                       limits = c(0, 1),
                       na.value = "grey") +
  labs(fill = "Proportion voting for a Democrat") +
  ggtitle("Results of the 2010 California State Assembly Election") +
  theme(panel.background = element_blank(),
        axis.title.x=element_blank(),
        axis.title.y=element_blank()) +
  scale_x_continuous(breaks = NULL) +
  scale_y_continuous(breaks = NULL)
HS10map
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-10-1.png)

``` r
HS12 <- Results %>%
  filter(year == 2012, chamber == "HS")

HS12map <- ggplot() +
  geom_map(data = HS12,
           aes(map_id = District, fill = perc_Dem),
           map = HSshapefort) +
  expand_limits(x = HSshapefort$long, y = HSshapefort$lat) +
  scale_fill_gradient2(low = muted("red"),
                       mid = "white", midpoint = .5,
                       high = muted("blue"),
                       limits = c(0, 1),
                       na.value = "grey") +
  labs(fill = "Proportion voting for a Democrat") +
  ggtitle("2012 results") +
  theme(panel.background = element_blank(),
        axis.title.x=element_blank(),
        axis.title.y=element_blank()) +
  scale_x_continuous(breaks = NULL) +
  scale_y_continuous(breaks = NULL)
HS12map
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-10-2.png)

``` r
HS14 <- Results %>%
  filter(year == 2014, chamber == "HS")

HS14map <- ggplot() +
  geom_map(data = HS14,
           aes(map_id = District, fill = perc_Dem),
           map = HSshapefort) +
  expand_limits(x = HSshapefort$long, y = HSshapefort$lat) +
  scale_fill_gradient2(low = muted("red"),
                       mid = "white", midpoint = .5,
                       high = muted("blue"),
                       limits = c(0, 1),
                       na.value = "grey") +
  ggtitle("2014 results") +
  labs(fill = "Proportion voting for a Democrat") +
  theme(panel.background = element_blank(),
        axis.title.x=element_blank(),
        axis.title.y=element_blank()) +
  scale_x_continuous(breaks = NULL) +
  scale_y_continuous(breaks = NULL)
HS14map
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-10-3.png)

``` r
HS16 <- Results %>%
  filter(year == 2016, chamber == "HS")

HS16map <- ggplot() +
  geom_map(data = HS16,
           aes(map_id = District, fill = perc_Dem),
           map = HSshapefort) +
  expand_limits(x = HSshapefort$long, y = HSshapefort$lat) +
  scale_fill_gradient2(low = muted("red"),
                       mid = "white", midpoint = .5,
                       high = muted("blue"),
                       limits = c(0, 1),
                       na.value = "grey") +
  labs(fill = "Proportion voting for a Democrat") +
  ggtitle("2016 results") +
  theme(panel.background = element_blank(),
        axis.title.x=element_blank(),
        axis.title.y=element_blank()) +
  scale_x_continuous(breaks = NULL) +
  scale_y_continuous(breaks = NULL)
HS16map
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-10-4.png)

``` r
# Visualizing the relationship between Approve*prez_party and perc_Dem

Results <- Results %>%
  mutate(prez_partyname = ifelse(prez_party == 1, "Democrat", "Republican"))

plot3 <- ggplot(Results, aes(y = perc_Dem, x = Approve)) +
  geom_smooth(method = "lm", aes(col = prez_partyname), se = FALSE) +
  geom_jitter() +
  xlim(.3, .7) +
  ylim(.3, .7) +
  scale_colour_manual(values = c("blue", "red")) +
  ylab("Proportion of votes won by Democrat") +
  xlab("National presidential approval rating") +
  labs(col = "President's party") +
  ggtitle("Voting behavior vs presidential approval rating")
plot3
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-11-1.png)

Create a Mixed Effects Model
----------------------------

``` r
# We need to code for NAs so that the observations used in our model will not
# be excluded if one of the variables we use include an NA. We will store these
# changes under a new dataset Resultsmodel. Also, we will code for lagged
# variables, which include observations for each variable from the previous
# election.
Results["perc_prez_Dem"][is.na(Results["perc_prez_Dem"])] <- 0
Results <- Results %>%
  mutate(perc_changeLag = lag(perc_change, 120),
         ApproveLag = lag(Approve, 120),
         incumbentpartyLag = lag(incumbentparty, 120),
         perc_DemLag = lag(perc_Dem, 120),
         perc_prez_DemLag2 = (lag(perc_prez_Dem, 240) +
                                2*lag(perc_prez_Dem, 120))/3,
         perc_gub_DemLag = lag(perc_gub_Dem, 120),
         perc_sen_DemLag = lag(perc_sen_Dem, 120),
         Approve = Approve*prez_party,
         chamdist = rep(1:120, times = 22))
Results["perc_prez_DemLag2"][is.na(Results["perc_prez_DemLag2"])] <- 0
Resultsmodel <- Results



Resultsmodel <- Resultsmodel %>%
  mutate(perc_prez_DemLag2 = (lag(perc_prez_Dem, 240) +
                                2*lag(perc_prez_Dem, 120))/3,
         perc_change = perc_change*prez_party)
```

New Model
---------

``` r
# Still need chamdist to indentify unique districts

Resultsmodel <- Resultsmodel %>%
  mutate(Approve = prez_party*Approve,
         prez_elec = ifelse(year %in% c(1976, 1980, 1984, 1988, 1992, 1996,
                                        2000, 2004, 2008, 2012, 2016), 1, 0),
         p_prez_Dem_elec = perc_prez_Dem*prez_elec)
```

``` r
# Models ranging from simple to complex
m1 <- lmer(perc_Dem ~
             incumbentparty +
             perc_change +
             (1|chamdist),
           data = Resultsmodel)

m2 <- lmer(perc_Dem ~
             Approve +
             incumbentparty +
             perc_change +
             (1|chamdist),
           data = Resultsmodel)

m3 <- lmer(perc_Dem ~
             perc_DemLag +
             Approve +
             incumbentparty +
             perc_change +
             (1|chamdist),
           data = Resultsmodel)

# m3 added a lagged vote for legislative offices the previous year, but this is
# not looking like a better model. The coefficient for this variable was
# negative which is not what we expect. One reason this variable might not
# be a good predictor is because many observations have NAs.

m4 <- lmer(perc_Dem ~
             mid_penalty +
             Approve +
             incumbentparty +
             perc_prez_DemLag2 +
             (1|chamdist),
           data = Resultsmodel)
# model 4 adds a weighted variable measuring prior voting behavior for the
# presidential office

m5 <- lmer(perc_Dem ~
             mid_penalty +
             Approve +
             incumbentparty +
             perc_prez_DemLag2 + 
             perc_sen_DemLag +
             (1 + perc_sen_DemLag|Approve) +
             (1|chamdist),
           data = Resultsmodel)

# Tried Adding a term where perc_sen_DemLag is tied to the Approval Rating--led to seemingly better results than m4, but my concern is that midterm penalty already accounts for the effect that I'm looking for (which is that a lower or worse approval rating will decrease the slope for perc_sen_DemLag). However, t value is low for perc_sen_DemLag and perc_prez_DemLag2, so let's try again...

m6 <- lmer(perc_Dem ~
             mid_penalty +
             Approve +
             incumbentparty +
             perc_prez_DemLag2 +
             perc_sen_DemLag +
             (1|chamdist),
           data = Resultsmodel)

# perc_prez_DemLag2 always seems to have a low t-value

m7 <- lmer(perc_Dem ~
             mid_penalty +
             Approve +
             incumbentparty +
             perc_prez_DemLag2 +
             perc_change +
             (1|chamdist),
           data = Resultsmodel)

# Adding perc_gub_DemLag as a fixed effect decreases the t-value of Approve.

m8 <- lmer(perc_Dem ~
             mid_penalty +
             incumbentparty +
             perc_sen_DemLag +
             (1|chamdist),
           data = Resultsmodel)
summary(m8)
```

    ## Linear mixed model fit by REML ['lmerMod']
    ## Formula: perc_Dem ~ mid_penalty + incumbentparty + perc_sen_DemLag + (1 |  
    ##     chamdist)
    ##    Data: Resultsmodel
    ## 
    ## REML criterion at convergence: 574.3
    ## 
    ## Scaled residuals: 
    ##     Min      1Q  Median      3Q     Max 
    ## -1.9008 -0.6554  0.1190  0.6470  1.7071 
    ## 
    ## Random effects:
    ##  Groups   Name        Variance Std.Dev.
    ##  chamdist (Intercept) 0.003337 0.05777 
    ##  Residual             0.101579 0.31871 
    ## Number of obs: 957, groups:  chamdist, 120
    ## 
    ## Fixed effects:
    ##                 Estimate Std. Error t value
    ## (Intercept)      0.43504    0.04686   9.285
    ## mid_penalty     -0.02445    0.01459  -1.676
    ## incumbentparty   0.27936    0.01667  16.757
    ## perc_sen_DemLag  0.13329    0.08159   1.634
    ## 
    ## Correlation of Fixed Effects:
    ##             (Intr) md_pnl incmbn
    ## mid_penalty -0.088              
    ## incmbntprty  0.359 -0.130       
    ## prc_sn_DmLg -0.967  0.106 -0.412

``` r
#Simpler model but t-values are higher
```

``` r
# Building a function to find MSE

Calc_err <- function(formula = perc_Dem ~
             mid_penalty +
             incumbentparty +
             perc_sen_DemLag +
             (1|chamdist),
             data = train,
             initial_lag = 12){
  n <- length(unique(Resultsmodel$year))
  n_predict <- n - initial_lag -1
  RSS <- rep(NA, times = n_predict)
  for(i in 1:n_predict){
    train <- Results %>%
      filter(year < min(Resultsmodel$year) + 2*(initial_lag + i) - 1)
    test <- Results %>%
      filter(year == min(Resultsmodel$year) + 2*(initial_lag + i))
    m <- lmer(formula = perc_Dem ~
             mid_penalty +
             incumbentparty +
             perc_sen_DemLag +
             (1|chamdist),
               data = train)
    y1 <- predict(m , data = test, na.rm = TRUE)
    y2 <- simulate(m , data = test, na.rm = TRUE)
    RSS[i] <- sum((test$perc_Dem - y2)^2, na.rm = TRUE)
  }
  mean(RSS)}

Calc_err(perc_Dem ~
             mid_penalty +
             incumbentparty +
             perc_sen_DemLag +
             (1|chamdist))
```

    ## [1] 126.9437

### Using the model to predict 2018 results

``` r
Resultsfin <- full_join(Resultsmodel, df2018, by = c("year",
                                             "chamber",
                                             "chamdist",
                                             "prez_party",
                                             "incumbentparty",
                                             "mid_penalty"))

Resultsfin[c("incumbentparty",
             "perc_sen_DemLag")][is.na(Resultsfin[c("incumbentparty",
                                                     "perc_sen_DemLag")])] <- 0

Resultsfin$sim <- simulate(m8,
                           seed = 23,
                           re.form=NA,
                           allow.new.levels = TRUE,
                           newdata = Resultsfin)$sim_1

Results2018 <- Resultsfin %>%
  filter(year == 2018,
         chamdist %in% c(1:80, 82, 84, 86, 88, 90, 92, 94, 96, 98, 100, 102, 104, 106,
                         108, 110, 112, 114, 116, 118, 120))

Dems2018 <- Results2018 %>%
  filter(sim >.5,
         Legparty == -1)
Reps2018 <- Results2018 %>%
  filter(sim<.5,
         Legparty == 1)
bp <- ggplot(Results2018) +
  geom_boxplot(aes(y = sim, x = as.factor(incumbentparty)))
bp
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-16-1.png)

``` r
# bp is a boxplot of our predicted results grouped by incumbency
```

``` r
# Importing the last remaining senatorial districts for map plot
lastresults <- read_excel("../../data-raw/2018LastDistricts.xlsx")
```

### 2018 maps

``` r
HS18 <- Results2018 %>%
  filter(year == 2018,
         chamber == "HS") %>%
  mutate(District = chamdist)
HS18$sim[HS18$sim<0] <- 0
HS18$sim[HS18$sim>1] <- 1

HS18map <- ggplot() +
  geom_map(data = HS18,
           aes(map_id = District, fill = sim),
           map = HSshapefort) +
  expand_limits(x = HSshapefort$long, y = HSshapefort$lat) +
  scale_fill_gradient2(low = muted("red"),
                       mid = "white", midpoint = .5,
                       high = muted("blue"),
                       limits = c(0, 1),
                       na.value = "grey") +
  labs(fill = "Proportion voting for a Democrat") +
  ggtitle("Forecasting 2018 assembly results") +
  theme(panel.background = element_blank(),
        axis.title.x=element_blank(),
        axis.title.y=element_blank()) +
  scale_x_continuous(breaks = NULL) +
  scale_y_continuous(breaks = NULL)
HS18map
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-18-1.png)

``` r
SEN18 <- Results2018 %>%
  filter(year == 2018,
         chamber == "SEN") %>%
  mutate(District = chamdist - 80)
SEN18$sim[SEN18$sim<0] <- 0
SEN18$sim[SEN18$sim>1] <- 1
SEN18 <- full_join(SEN18, lastresults, by = c("year",
                                              "chamber",
                                              "District"))

SEN18map <- ggplot() +
  geom_map(data = SEN18,
           aes(map_id = District, fill = sim),
           map = HSshapefort) +
  expand_limits(x = HSshapefort$long, y = HSshapefort$lat) +
  scale_fill_gradient2(low = muted("red"),
                       mid = "white", midpoint = .5,
                       high = muted("blue"),
                       limits = c(0, 1),
                       na.value = "grey") +
  labs(fill = "Proportion voting for a Democrat") +
  ggtitle("Forecasting 2018 senatorial results") +
  theme(panel.background = element_blank(),
        axis.title.x=element_blank(),
        axis.title.y=element_blank()) +
  scale_x_continuous(breaks = NULL) +
  scale_y_continuous(breaks = NULL)
SEN18map
```

![](FinalProject_files/figure-markdown_github/unnamed-chunk-18-2.png)
