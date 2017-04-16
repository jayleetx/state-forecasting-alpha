Forecasting the 2018 California Legislative Election
================
EJ Arce
4/2/2017

### Import

We are importing three datasets and we will eventually join them. LegResults6814 consists of every candidate who ran for a seat in the California State Legislature from 1968 to 2014. LegResults16 includes such candidates who ran in 2016. Threeoffices includes the voting totals for three US offices (Presidential, Gubernatorial, and US Senate) in each of the California senatorial and assembly districts, from 1966 to 2016. This dataset previously consisted of three separate datasets, which Jonathan and I cleaned in R or excel, to create threeoffices.

``` r
LegResults6814 <- read.dta("032_StateLegForecast_CAcopy/SLERs1967to2015_20160912b_CA.dta")

LegResults16 <- read_excel("032_StateLegForecast_CAcopy/001_CA2016GenOfficecopy.xls")

threeoffices <- read_excel("032_StateLegForecast_CAcopy/threeoffices662016copy.xlsx")
```

### Wrangle and Tidy

For both datasets, we want each observation to be a given chamber's district's voting totals for both national and state legislative offices, where each office includes a democratic or republican candidate. It is certainly possible that some candidates for a legislative office ran unopposed or against a third-party candidate, resulting in some blank entries.

``` r
# Selecting and renaming variables of interest, and filtering the dataset to only include democratic and republican candidates.

LegResults6814 <- LegResults6814 %>%
  select(v05, v07, v09z, v21, v22, v23) %>%
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
LegResults6814$prez_party <- ifelse(LegResults6814$year %in% c(1968, 1978,1980,1994,1996,1998,2000,2010,2012,2014,2016), 1, -1)
LegResults6814$mid_penalty <- ifelse(LegResults6814$year %in% c(1968,1972,1976,1980,1984,1988,1992,1996,2000,2004,2008,2012), 0, ifelse(LegResults6814$year %in% c(1970,1974,1982,1986,1990,2002,2006), -1, 1))
LegResults6814 <- LegResults6814  %>%
  mutate(dummyparty = ifelse(party == "Dem", 1, -1)) %>%
  mutate(incumbent = dummyparty * incumbency) %>%
  spread(key = party, value = votes)
LegResults6814[is.na(LegResults6814)] <- 0



LegResults16 <- LegResults16 %>%
  select(v05, v07, v09, partyoriginal, v23, incumbency) %>%
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
  mutate(incumbent = dummyparty * incumbency) %>%
  mutate(prez_party = 1,
         mid_penalty = 0) %>%
  spread(key = party, value = votes)
LegResults16[is.na(LegResults16)] <- 0
```

``` r
# Aggregating rows by votes so that our LegResults 6814 dataset is now a district's legislative election

LegResults6814 <- aggregate(x = LegResults6814[c("Dem", "Repub", "incumbent")],
                            by = LegResults6814[c("year",
                                                  "chamber",
                                                  "District",
                                                  "prez_party",
                                                  "mid_penalty")],
                            sum)

LegResults16 <- aggregate(x = LegResults16[c("Dem", "Repub", "incumbent")],
                          by = LegResults16[c("year",
                                              "chamber",
                                              "District",
                                              "prez_party",
                                              "mid_penalty")],
                          sum)

# Selecting variables of interest for the database of results for 3 offices, removing unneccessary rows, and renaming the district column.

threeoffices <- threeoffices %>%
  filter(Dummy1 == 1) %>%
  select(year, chamber, dist, (17:24)) %>%
  rename(District = dist)
```

### Join

``` r
LegResults <- full_join(LegResults16, LegResults6814, by = c("year", "chamber", "District", "prez_party", "mid_penalty", "Dem", "Repub", "incumbent"))
```

### Importing presidential approval and economy strength data

``` r
Econ_State <- read_excel("032_StateLegForecast_CAcopy/econ-strength.xls")
Prez_Approv <- read_excel("032_StateLegForecast_CAcopy/Gallup_Prez_Approv_1937to2014_2014_10_22.xlsx")

Prez_Approv <- Prez_Approv %>%
  filter(start_year >= "1968")

Econ_State <- Econ_State %>%
  spread(key = Quarter, value = Income) %>%
  mutate(I_IIpercent = (II-I)/I,
         II_IIIpercent = (III-II)/II,
         III_IVpercent = (IV-III)/III,
         IV_Ipercent = (lead(I) - IV)/IV) %>%
  mutate(perc_change = (4*I_IIpercent +
                        3*IV_Ipercent +
                        2*III_IVpercent +
                        IV_Ipercent)/10)
```

### Joining legislative elections data with threeoffices

``` r
Results <- full_join(LegResults, threeoffices, by = c("year", "chamber", "District"))
Results <- Results %>%
  rename(Leg_Dem = Dem, Leg_Repub = Repub) %>%
  mutate(incumb_party = ifelse(incumbent == 1, "Democrat", ifelse(incumbent == 0, "No incumbent", "Republican")))
```

### Viewing Clean Data

``` r
View(Results)
```

### Some visualizations

``` r
plot1 <- ggplot(Results) +
  geom_point(mapping = aes(y = Leg_Dem, x = Leg_Repub, col = incumb_party)) +
  scale_colour_manual(values = c("blue", "black", "red"))
```