# Reproducible Research: Peer Assessment 1
iNLyze  


## Loading and preprocessing the data
First we need to unzip. For simplicity, let's assume the data is in the current directory (as it would be in a fresh fork of the project). 


```r
unzip("activity.zip")
```

I like to use dplyr and lubridate, so let's require those libraries. While we are at it attach ggplot2 as well. 


```r
if (!require(dplyr)) install.packages('dplyr')
```

```
## Loading required package: dplyr
## 
## Attaching package: 'dplyr'
## 
## The following object is masked from 'package:stats':
## 
##     filter
## 
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
library(dplyr)
if (!require(lubridate)) install.packages('lubridate')
```

```
## Loading required package: lubridate
```

```r
library(lubridate)
if (!require(ggplot2)) install.packages('ggplot2')
```

```
## Loading required package: ggplot2
```

```r
library(ggplot2)
```

Done. Now we shall load the data using read.csv(). It is also converted into a data.table using tbl_df()


```r
activity <- read.csv("activity.csv", header = TRUE, stringsAsFactors = FALSE, sep = ",")
activity <- tbl_df(activity)
```



The dates must be in POSIX format. We'll use lubridate for that and confirm using str()


```r
activity$date <- ymd(activity$date)
str(activity)
```

```
## Classes 'tbl_df', 'tbl' and 'data.frame':	17568 obs. of  3 variables:
##  $ steps   : int  NA NA NA NA NA NA NA NA NA NA ...
##  $ date    : POSIXct, format: "2012-10-01" "2012-10-01" ...
##  $ interval: int  0 5 10 15 20 25 30 35 40 45 ...
```



## What is mean total number of steps taken per day?
I am grouping the data by "date" and summarize() using dplyr. I am keeping the plot simple, no fancy formatting. Hope you'll excuse. 


```r
activity <- group_by(activity, date)
q1 <- summarize(activity, steps.per.day = sum(steps[complete.cases(steps)]))
qplot(steps.per.day, data=q1)
```

```
## stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust this.
```

![](README_files/figure-html/Q1-1.png) 

```r
mean.steps <- with(q1, mean(steps.per.day[complete.cases(steps.per.day)]) )
median.steps <- with(q1, median(steps.per.day[complete.cases(steps.per.day)]) )
```

The mean number of steps taken per day is 9354.2295082 and the median of the number of steps taken per day is 10395.


## What is the average daily activity pattern?

Here I use lubridate again to create the time interval in POSIX format. Unfortunately, I didn't find a lubridate function I could feed the interval variable to. Thus I wrote this little accessory function:

```r
## Convert interval variable to POSiX in one go 
timewarp <- function(x) {
        int.sucker <- sprintf("%04d", x)
        int.colon <- paste(substr(int.sucker, 1, 2), substr(int.sucker, 3, 4), sep=":")
        #hm(x)
        int.colon
}
```

Now for the plot:
(Please note that I did convert "interval" to POSIX using today's date. I didn't  remove the Date part. It is plotted as a side-effect of the POSIX conversion, but I CAN plot gap-free. )

```r
## Ungroup
activity <- ungroup(activity)

## Set locale to plot time in English
Sys.setlocale("LC_TIME", "English")


## Create time interval in POSIXct format and use it for plotting
q2 <- activity
dt <- timewarp(q2$interval)
dt <- as.POSIXct(dt, format = "%H:%M")
q2 <- mutate(q2, date_time = dt )

q2 <- group_by(q2, date_time)
vals <- complete.cases(q2)
q2.summary <- summarize(q2[vals,], daily.activity = mean(steps))
qplot(date_time, daily.activity, data=q2.summary, geom="line")
```

![](README_files/figure-html/Q2-1.png) 

```r
# Find interval with maximal activity
max.activity <- max(q2.summary$daily.activity)
max.activity.index <- which(q2.summary$daily.activity == max.activity)
```

The maximal mean activity of 206.1698113 is found at . 

## Imputing missing values
Part 1 is easy - How many missing values do we have?


```r
number.nas <- sum(is.na(activity$steps))
```
The number of NAs is 2304

For the next part we first compute a new data table containing the mean number of steps per day. We will use this for interpolation. There are some days which have only Nas. We just set these to zero (i.e. no activity). 


```r
## Calculate mean activity per day
q3 <- group_by(activity, date)
q3 <- summarize(q3, mean.per.day = mean(steps[complete.cases(steps)]))

## Find NAs and fill them with zeros
there.are.NAs <- is.na(q3$mean.per.day)
if (sum(there.are.NAs)) {
     q3[there.are.NAs, ]$mean.per.day = 0  
}


## Do the imputing
activity.imputed <- left_join(activity, q3)
```

```
## Joining by: "date"
```

```r
activity.imputed$steps <- ifelse(is.na(activity.imputed$steps), activity.imputed$mean.per.day, activity.imputed$steps)
activity.imputed <- subset(activity.imputed, select=names(activity.imputed)[1:3])
```

Now that we have imputed the NAs, let's see what changed:


```r
activity.imputed <- group_by(activity.imputed, date)
q1.imputed <- summarize(activity.imputed, steps.per.day = sum(steps[complete.cases(steps)]))
qplot(steps.per.day, data=q1.imputed)
```

```
## stat_bin: binwidth defaulted to range/30. Use 'binwidth = x' to adjust this.
```

![](README_files/figure-html/Q3_04-1.png) 

```r
mean.steps.imputed <- with(q1.imputed, mean(steps.per.day[complete.cases(steps.per.day)]) )
median.steps.imputed <- with(q1.imputed, median(steps.per.day[complete.cases(steps.per.day)]) )

## Let's make a nice table for comparison
what.changed <- cbind(rbind(mean.steps, mean.steps.imputed), rbind(median.steps, median.steps.imputed))
what.changed <- as.data.frame(what.changed)
names(what.changed) <- c("Mean", "Median")
rownames(what.changed) <- c("Before Imputing", "After Imputing")
```

Now we can compare the differences:

```r
what.changed
```

```
##                    Mean Median
## Before Imputing 9354.23  10395
## After Imputing  9354.23  10395
```

And it turns out - there are no differences! Most likely this is because the Nas were groupded into days. That is, there were some days like, e.g. 2012-10-01, which contain only NAs, while the other days are complete.



## Are there differences in activity patterns between weekdays and weekends?

Note: we are using the imputed version of the activity data table.
Below you can see the differences in activity patterns during weekdays vs. weekends. 


```r
## Create new factor
all.days <- unique(weekdays(activity$date))
week.days <- all.days[1:5]
week.ends <- all.days[6:7]

day <- as.character(weekdays(activity.imputed$date))
q4 <- tbl_df(cbind(activity.imputed, day))
q4$day <- as.factor(ifelse(q4$day %in% week.ends, "weekend", "weekday" ))

##Now for the plot
dt <- timewarp(q4$interval)
dt <- as.POSIXct(dt, format = "%H:%M")
q4 <- mutate(q4, date_time = dt )

q4 <- group_by(q4, day, date_time)
vals <- complete.cases(q4)
q4.summary <- summarize(q4[vals,], daily.activity = mean(steps))
qplot(date_time, daily.activity, data=q4.summary, geom="line", facets=day~.)
```

![](README_files/figure-html/Q4-1.png) 

