---
layout: post
title: Example Mapping Application
subtitle: A step by step guide on how to create a mapping applicatin using R
published: true
date: '2017-03-29'
---

[Launch  Example Mapping Application](http://aebarros.com/shiny/CatchApp/ExampleMapApp/)
![screenshot of app](/img/examplemapappscreen.png)
[Github link](https://github.com/aebarros/shiny-server/tree/master/ExampleMapApp)

---

## Introduction

This post will act as a step by step guide on how to use R to create basic mapping application for spatial data.  
This guide assumes a basic understanding of R, [RStudio](https://www.rstudio.com/), and [R Shiny](https://shiny.rstudio.com/).
The guide will be utilizing the following packages:

* [plyr](https://cran.r-project.org/web/packages/plyr/plyr.pdf) 
* [dplyr](https://cran.r-project.org/web/packages/dplyr/dplyr.pdf) 
* [lubridate](https://cran.r-project.org/web/packages/lubridate/lubridate.pdf) 
* [reshape2](https://cran.r-project.org/web/packages/reshape2/reshape2.pdf) 
* [purr](https://cran.r-project.org/web/packages/purrr/purrr.pdf) 
* [shiny](https://cran.r-project.org/web/packages/shiny/shiny.pdf) 
* [shinythemes](https://cran.r-project.org/web/packages/shinythemes/shinythemes.pdf) 
* [leaflet](https://cran.r-project.org/web/packages/leaflet/leaflet.pdf) 
* [rsconnect](https://cran.r-project.org/web/packages/rsconnect/rsconnect.pdf) 

You may use whatever set of data you like along with this guide, as long as it has some key variables:

1. the data you wish to display
2. latitude and longitude coordinates
3. date/time information

We will be utilizing catch information from the California Department of Fish and Wildlifes 20mm net survey. This survey uses a 20mm sled to target juvenile smelt in the Sacramento/San Joaquin delta during the spring months.

---

##Collecting Our Data

For this guide I used the 20mm catch data from [CDFW](http://www.dfg.ca.gov/delta/projects.asp?ProjectID=20mm). I downloaded the access database
from [ftp://ftp.dfg.ca.gov/Delta%20Smelt/] labeled as "20-mm.mdb". The 20 mm data base has five tables that we will be using:

* 20 mm stations (contains a list of station names and gps coordinates)
* Catch (tells us the catch of each species of fish for each tow using a fishcode, this is what we wish to display)
* Fish Codes (links the fishcodes to common names for each species)
* Tow info (links the catch info to a specific tow, station, date, and duration)

I exported each of these tables as a tab delimited .txt file into a "data" folder within my R-project.

---

##Cleaning Our Data

In order to make the mapping application I need to bring together all the data from the above tables and get it into a format that I 
can work with. My end goal here is a dataset similar to the following:

Station|Date|longitude|latitude|Common.Name|CPUE
---|---|---|---|---|---
809|1995-04-24|-121.6892|38.05250|American shad|0

(note: catch per unit effort (CPUE) is the data we will be displaying)

In order to reach this point we will be creating a cleaning script titled "global.R" saved within our project.
