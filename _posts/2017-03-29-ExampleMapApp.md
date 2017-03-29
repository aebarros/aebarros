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

This post will act as a step by step guide on how to use R to create basic mapping application for spatial data. This guide assumes a basic understanding of R, [RStudio](https://www.rstudio.com/), and [R Shiny](https://shiny.rstudio.com/).
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
All the scripts and data for this guide can be found [here](https://github.com/aebarros/shiny-server/tree/master/ExampleMapApp)

---

## Collecting Our Data

For this guide I used the 20mm catch data from [CDFW](http://www.dfg.ca.gov/delta/projects.asp?ProjectID=20mm). I downloaded the access database
from [ftp://ftp.dfg.ca.gov/Delta%20Smelt/] labeled as "20-mm.mdb". The 20 mm data base has four tables that we will be using:

* 20 mm stations (contains a list of station names and gps coordinates)
* Catch (tells us the catch of each species of fish for each tow using a fishcode, this is what we wish to display)
* Fish Codes (links the fishcodes to common names for each species)
* Tow info (links the catch info to a specific tow, station, date, and duration)

I exported each of these tables as a tab delimited .txt file into a "data" folder within my R-project.

---

## Cleaning Our Data

In order to make the mapping application I need to bring together all the data from the above tables and get it into a format that I 
can work with. My end goal here is a dataset similar to the following:

Station|Date|longitude|latitude|Common.Name|CPUE
---|---|---|---|---|---
809|1995-04-24|-121.6892|38.05250|American shad|0

(note: catch per unit effort (CPUE) is the data we will be displaying)

If your data already looks like this, you can skip this section entirely.

In order to reach this point we will be creating a cleaning script titled "global.R" saved within our project.
We being the script by loading our packages:

```R
library(plyr)
library(dplyr)
library(lubridate)
library(reshape2)
library(purrr)
```

followed by loading the required data:

```R
data.catch=read.table("data/catch.txt", header= T, sep= "\t")
fishcodes=read.table("data/fishcodes.txt", header= T, sep= "\t")
data.stations=read.table("data/stations.txt", header= T, sep= "\t")
data.tows=read.table("data/towinfo.txt", header= T, sep= "\t")
```

Next I want to inspect the elements using the sapply funtion. This funtion is useful to discover the class/type of each vector.

```R
#####inspect elements#
head(data.stations)
sapply(data.stations,class)
```

After running this code you may notice some obvious issues with our data:

1. Latitude and Longitude are each broken up into seperate vectors for degrees, minutes, and seconds
2. Each of those vectors has a different class, either integer or numeric

This is not at all what we want, so our first step will be to turn our Latitude and Longitude into decimal degree format, and each under its own vector using the following calls:

```R
#transform into decimal degrees
data.stations$LatS<-data.stations$LatS/3600
data.stations$LatM<-data.stations$LatM/60
data.stations$LonS<-data.stations$LonS/3600
data.stations$LonM<-data.stations$LonM/60
#add minutes and seconds together
data.stations$LatMS<-data.stations$LatM+data.stations$LatS
data.stations$LonMS<-data.stations$LonM+data.stations$LonS
#combine data
data.stations$Latitude <- paste(data.stations$LatD,data.stations$LatMS)
data.stations$Longitude <- paste(data.stations$LonD,data.stations$LonMS)
head(data.stations)
#remove spaces and first zero, replace with "."
data.stations$Latitude <- gsub(" 0.", ".", data.stations$Latitude)
data.stations$Longitude <- gsub(" 0.", ".", data.stations$Longitude)
head(data.stations)
#add "-" infront of all longitude
data.stations$negative <- rep("-",nrow(data.stations)) # make new column 
data.stations$Longitude<-paste(data.stations$negative, data.stations$Longitude)
data.stations$Longitude <- gsub(" ", "", data.stations$Longitude)
head(data.stations)
#keep columns we need#
keep<-c("Station","Latitude","Longitude")
data.stations<-data.stations[ , which(names(data.stations) %in% keep)]
head(data.stations)
#transform Lat and Long to numeric class
data.stations<-transform(data.stations, Latitude = as.numeric(Latitude), Longitude = as.numeric(Longitude))
```

Now a simple `head(data.stations)` call will show us that at least our stations information is formated properly.

Our next step in preparing our data for the application is to joing all our tables together using a series of `inner_join` calls. I do this here with dplyr piping "%>%".

```R
data<-data.catch%>%
  inner_join(data.stations)%>%
  inner_join(data.tows)%>%
  inner_join(fishcodes)
```

Using `head(data)` and `sapply(data,class)` calls we can see that the joined data tables has a load of vectors we don't need, as well as
 the Date vector having a class of "factor". We need the Date vector to have a "date" class, and to remove the unnecessary columns like so:
 
```R
#format dates#
sapply(data,class)
data$Date <- as.Date(data$Date , "%m/%d/%Y")

#keep columns we need#
keeps<-c("Date","Station","Latitude","Longitude","Fish.Code","Catch","Duration","Common.Name")
data<-data[ , which(names(data) %in% keeps)]
```

Next I want to do two things, first calculate a Catch Per Unit of Effort (here calculated as catch/tow duration) and then add in an "All" option which combines the catch for all species caught in each tow. This is all done using the following:

```R
#calculate CPUE
data$CPUE=data$Catch/data$Duration
head(data)

###Next section to calculate "All" CPUE for each tow
#reshape fish catch to wide format
datawide <- dcast(data, Station +Latitude+Longitude+Duration+ Date ~ Common.Name, value.var="CPUE",fun.aggregate=sum)
head(datawide)
#learn which column numbers apply to fish
which(colnames(datawide)=="American shad")
which(colnames(datawide)=="yellowfin goby")
#calculate All
datawide$All=rowSums(datawide[,6:71], na.rm=T)
head(datawide)
#melt back straight
data=melt(datawide,id.vars=c("Station","Date","Longitude","Latitude"), measure.vars=c(6:72),
             variable.name="Common.Name",
             value.name="CPUE")
```

The dcast and melt calls used above are both from the reshape2 package, and restructure a molten data frame into an array or data frame.

Finally we want to change the vector name for the Latitude and Longitude vectors in our data frame. When creating the application, the leaflet package will automatically recognize the lat and long vectors as coordinates to use, but not if they are named "Latitude" and "Longitude" with capital "L"s. Here we simply change those to lower case to make life easier down the road.

```R
#rename Lat and Lon for leaflet
data<-rename(data, latitude=Latitude, longitude=Longitude)
```

Now our data is clean and ready to use!

##The application page here will be constucted in due time

