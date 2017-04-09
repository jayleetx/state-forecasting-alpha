Import
================
EJ Arce
4/2/2017

### Import

``` r
LegResults <- read.dta("032_StateLegForecast_CAcopy/SLERs1967to2015_20160912b_CA.dta")
threeoffices9008 <- read_excel("032_StateLegForecast_CAcopy/aaa_All_States_1968_2010_2013_03_22_CAonly.xlsx")
threeoffices6688 <- read_excel("032_StateLegForecast_CAcopy/033_CA_1966to1988_20160715_FromNicole.xlsx")
```

### Tidy

``` r
# Selecting variables of interest (only old variables)
LegResults <- LegResults %>%
  select(caseid, v05, v07, v09z, 23:28, 30:33) %>%
  rename(year = v05,
         chamber = v07,
         District = v09z,
         termlength = v15,
         electiontype = v16)

# Selecting variables of interest for the database of results for 3 offices, recoding v07 to return SEN or HS, and renaming v07 as "chamber" for 9008

threeoffices9008$v07 <- ifelse(threeoffices9008$v07 == 8, "SEN", "HS")

threeoffices9008 <- threeoffices9008 %>%
  select(2:4, 6:9, 11, 12, 15, 16) %>%
  rename(year = v05, District = v09, chamber = v07)

threeoffices6688 <- threeoffices6688 %>%
  select(1, 3, 4, (17:25)) %>%
  rename(District = dist) %>%
  filter(Dummy1 == 1)
  
threeoffices6688$chamber <- ifelse(threeoffices6688$chamber == "S", "SEN", "HS")
```

``` r
# Joining threeoffices6688 and threeoffices9008 into threeoffices
threeoffices6608 <- full_join(threeoffices6688, threeoffices9008, key = "year")

# Create threeoffices1016
threeoffices1016 <- read_excel("032_StateLegForecast_CAcopy/California_2010_2016.xlsx")

# Join threeoffices6608 and threeoffices1016
threeoffices6616 <- full_join(threeoffices6608, threeoffices1016, key = "year")
```

### Load the Data for 2016 Results:

``` r
LegResults2016 <- read_excel("032_StateLegForecast_CAcopy/CA2016GenOffice.xls")
```

### Join the Datasets