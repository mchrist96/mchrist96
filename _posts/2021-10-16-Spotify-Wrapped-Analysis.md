# My Own Spotify Wrapped
## Intro
As a big fan of music and data analytics I decided to tackle a personal project that combines the two. The popular streaming service Spotify provides users with a year-end analysis of that user’s personalized listening trends called Wrapped. It is a popular occurrence every December and as a fan of music and data analytics I eagerly await mine every year. However, upon finding out that you can request your past year of personal data from Spotify, including your listening history, I decided to try to replicate wrapped as best as I could with the data they provide. The data Spotify provides includes, for each listen, time of occurrence, artist name, track name and the duration of the listen in milliseconds. Noticeable missing pieces are genre and track length, which are not provided, so for now I will not recreate the genre analysis normally provided in Wrapped and instead look at other aspects. I have decided to look at my top artists and tracks by number of plays as well as by amount of time spent which Spotify does not include on an artist or track level in their Wrapped. Additionally, I will recreate a few easy to calculate stats such as number of distinct artists and tracks listened to as well as a few new ones not included in Wrapped. 

When I set out to start this project I had the following questions I wanted to answer:
1) How long do I spend listening to a song on average?
2) How many different artists did I listen to over the past year?
3) How many different tracks did I listen to over the past year?
4) What were my top 10 artists?
5) What were my top 10 tracks?
6) Does my top 10 change when counting total time spent listening vs number of plays?
7) How fierce was the competition among my top 10 over the course of the year?
---
## Acquiring Spotify Data
The first step of this project was to acquire the Spotify data which can be done by going to your profile on the browser version of Spotify and going to privacy settings. From here there is an option to request your data which it says can take up to 30 days and will be emailed to you for download when ready.

![Pic 1](https://user-images.githubusercontent.com/92491074/137600609-300865c2-64e8-4f54-9d50-cb38e02727ea.png)

In my experience it only took roughly 3 days before the download link arrived in my inbox. From here use the download link and get your data. The data is provided in a zip file which contains many JSON files and one HTML file that redirects to the Understanding My Data article on Spotify’s site which gives a breakdown of what all is included. There is a lot of other data included such as your personal information and billing information as well as playlists and search history. For our purposes we need the file(s) labelled StreamingHistory#. There will likely be multiple of these files numbered starting from 0. Each file contains 10,000 streams except for the last file which will have a remainder amount. In my case there were 4 files numbered 0 to 3 representing ~30,600 total streams over the past year.

![Pic 2](https://user-images.githubusercontent.com/92491074/137600699-19efd9db-d24d-4500-bef1-5213ad0f1e26.png)

---
## Loading Data Into SQL
Next we have to load our data into a new SQL database (I created one called Spotify Data) and populate a new table with our data. Since the data is provided in JSON files we need to run the following code to load the JSON file contents into SQL and then break up the text into proper table format. 

```sql
Declare @JSON varchar(MAX)
Select @JSON=BulkColumn
From OPENROWSET (BULK 'FILE LOCATION', SINGLE_CLOB) import
Select * INTO ListeningHistory
From OPENJSON (@JSON)
With
(
[endTime] date,
[artistName] nvarchar(MAX),
[trackName] nvarchar(MAX),
[msPlayed] int
)
```

Within this code we specify the column names and assign data types to each. We do this for each file then insert all the data from tables 1-3 into table 0 and rename it ListeningHistory so we know what it is going forward. 

![Pic 5](https://user-images.githubusercontent.com/92491074/137601126-2899a8e1-9d77-4325-8504-2a6870534f2b.png)


---
## Cleaning/Initial Analysis (Stats)
Now that the data is all in one place where we can easily inspect the data I found that included in our data is podcasts which I wish to remove for this analysis so I can focus specifically on music. Removing the podcast entries proved to be simple as I had only listened to a few different podcasts (but many episodes of each) so I used the Delete statement and specified the 4 different podcast names using Where as shown below. 

```sql
Delete From ListeningHistory Where artistName in ('Podcast 1', 'Podcast 2', 'Podcast 3', 'Podcast 4')
```

Additionally, the data includes many entries where the msPlayed is extremely small representing 0 to a few seconds which I will take as being I skipped the song. I do not wish for these entries to be included in the analysis since I did not really listen to the song. To determine the cutoff point I used the track length of the shortest song I knew I had listened to which from memory was a 10 second mini punk song by a band I had seen live. Thus I decided to remove all entries where msPlayed was less than 10000 (this converts to 10.000 seconds). What remained was the data I wished to use for all analysis going forward. 

```sql
Delete From ListeningHistory Where msPlayed <10000
```

What is now left is the data I wish to use in my analysis. The first three questions I had can easily be answered using basic SQL quereies.

To find out how many different artists and tracks I listened to and how long I listened per play on average (in seconds) we can run the following query.
```sql
select count(distinct(artistName)) as Unique_artists, count(distinct(trackName)) as Unique_tracks, AVG(convert(bigint, msplayed)/1000) as Avg_Time_Played_Sec
from ListeningHistory
```

The results from this query are that I listened to `1125` unique artists, `5088` unique tracks, and listened for `145 seconds` per play on average.

---
## Establishing RODBC Connection between R and SQL
For the remaining more complex aspects of my analysis I wished to use R instead of SQL so it was necessary to import my data into R. The way I decided to do this was use the RODBC package in R to connect to my SQL database. The benefits of this method are ease and speed. Establishing a connection is rather simple and going forward it is possible to use the SQLQuery() function to write SQL queries in R and assign the output directly to R objects all without ever leaving RStudio. The code below shows how this is performed.

```r
library(RODBC)
con <- odbcConnect("Spotify Data")
Spotify_Data <- sqlQuery(con,
"Select *
From ListeningHistory")
```

Now we can open the object Spotify_Data in R and see all the listening history plays from SQL that we now have access to. 

![Pic 7](https://user-images.githubusercontent.com/92491074/137601338-ad26b3f7-8d50-4514-8173-bf57e405b3c3.png)

---
## Top 10 Artists
Now that we have access to our SQL tables in R we can perform some more complex calculations. The first thing I wanted to find was my top artists by number of plays. This can be done very easily in SQL but I want to take it a step further and calculate the cumulative plays throughout the course of the year so that I can plot my top 10 artists competed over the year. To start I ran the following SQL query to get the names of my top 10 artists by plays and the corresponding number of total plays.

```r
library(ggplot2)
Date <- seq(as.Date('2020-10-01'), as.Date('2021-10-02'), by=1)

top_artists <- sqlQuery(con, 
"SELECT TOP(10) artistName, count(*) as Total
From listeninghistory 
Group by artistName 
Order by Total desc; ")
barplot(top_artists[,2], names.arg = top_artists[,1], cex.names = .5, ylab = 'Plays', xlab = 'Artist', las = TRUE, ylim = c(0,1500))
```

To start I took the names of the top 10 artists from the SQL query and converted them into a vector so that I can use in a future for loop. Next I created an empty data frame that was 10 columns by 366 rows. The columns represent each of the top 10 artists and the columns are for the days. I added a prior day of all 0s as a baseline start for my for loop since the cumulative calculations counts the plays for a given day and adds them to the previous day. So for day 1 we total the number of plays for each of the top ten artists and add them to 0 (since this is the beginning of the cumulative calculation) then for day 2 it does the same but adds the day 1 value and so on through the end of the year. This for loop is shown below.

```r
top_artists <- top_artists[,-2]
top_artists <- as.vector(t(top_artists))


Cumulative <- matrix(ncol = 10, nrow = 365)
Cumulative[1,] = 0
Cumulative <- data.frame(Cumulative)
colnames(Cumulative) <- top_artists


for(i in 2:length(Date)){
  for(j in 1:length(top_artists)){
    Cumulative[i,j] = sum(SpotifyData$endTime == Date[i] & SpotifyData$artistName == top_artists[j]) + Cumulative[i-1,j]
  }
}
```

Then I plotted the data frame to see how the artists competed over the past year for their spot in my top 10. To do this, and all the following line plots, I used the `reshape` package to melt my data frame and also appended the date column to make it easier to plot in ggplot2.

![Cumulative Plays (Artists)](https://user-images.githubusercontent.com/92491074/137603326-a57a45c7-c991-472c-b2a0-70e9694dd905.jpeg)

When looking at my previous plots I realized this might not give the full story of my listening habits. For instance my number one artist of the year was a rap artist whose songs are much shorter on average than a metal artist further down in my top 10. So I decided to recreate the prior cumulative calculations but this time for total time spent listening instead of total plays. I performed a different SQL query shown below to capture my top 10 artists by total time. Then I went through the same steps as before but with a new for loop shown below.

```r
top_artists_time <- sqlQuery(con, 
"Select distinct top(10) artistname, sum(msplayed) as Total_Time
From listeninghistory
Group by artistname
Order by total_time desc")

top_artists_time <- top_artists_time[,-2]
top_artists_time <- as.vector(t(top_artists_time))

Cumulative_time <- matrix(ncol=10, nrow=365)
Cumulative_time[1,] = 0
Cumulative_time <- data.frame(Cumulative_time)
colnames(Cumulative_time) <- top_artists_time

for(i in 2:length(Date)){
  for(j in 1:length(top_artists_time)){
    Cumulative_time[i,j] = sum(SpotifyData[which(SpotifyData$endTime == Date[i] & SpotifyData$artistName == top_artists_time[j]), 4]) + Cumulative_time[i-1,j]
  }
}
```

This resulted in the following plot which shows a very different story than the version that counted plays instead of time. For instance `Agalloch` a metal band is number 1 (albeit barely) whereas previously they were in spot 7. Further, `Tony Molina` and `Fall Out Boy` both fall out of the top 10 artists, while `maudlin of the Well` and `Deafheaven` both metal bands, enter the top 10.

![Cumulative Time (Artists)](https://user-images.githubusercontent.com/92491074/137603432-4fd4c600-ad30-42f6-b6e3-6d144fbb4bd8.jpeg)

---
## Top 10 Tacks

I repeated the analysis from above but this time instead of artists I performed a query for top 10 tracks by plays. The following is the resulting plot for top 10 tracks.

![Cumulative Plays (Tracks)](https://user-images.githubusercontent.com/92491074/137603344-1e381973-7917-4ea5-9a71-fd2d8ac8f20f.jpeg)

Then I repeated my alterations to the loop for calculating based on time and reran for top 10 tacks based on time. The following is the resulting plot.

![Cumulative Time (Tracks)](https://user-images.githubusercontent.com/92491074/137603351-9cdcc47b-f8b4-4195-b213-b49c79b03afa.jpeg)

---
## Findings
Our findings show us that when looking at top artists the results can, and in my case do, differ significantly based on whether or not the calculation is made using number of plays or the amount of time spent listening. The reason for this, in my data, was that a significant portion of my musical taste is metal music which on average has a much longer track length while on the flip side I also listen to a lot of rap and punk both of which have much shorter track lengths on average. To show an example, in a 10 minute span I could listen to a 2.5 minute rap song 4 times whereas some of my most listened to metal songs have track lengths of well over 10 minutes and would not even have a full play in that time span. Thus it is much easier to rack up a high number of plays for genres with lower average track lengths, assuming you are listening to the entire song or at least a substantial part of the song. This trend is even more evident when looking at the track analysis instead of the artist as when looking at tracks by time 7 of the 10 songs are long(4.5 min+) metal tracks but when counting by plays the top 10 tracks are overwhelmingly short pop and rap tracks. 

---
## Future Analysis Directions
In the future it would be interesting to perform web scraping to gather the genre classification for each track and perform a genre analysis of the sort that is normally included in Spotify’s Wrapped. Additionally, I discovered at one point in my data that a 4.5 minute song was had an entry where the msPlayed corresponded to over 10 minutes. Upon further digging I concluded that when the progress slider on the Spotify app is slid backwards in the direction of, but not all the way to, the start of the track the entry is not ended but continued and all subsequent time is added to the pre slide time resulting in some listens that are longer than the track length. If track length data were included it would be interesting to calculate and count plays by non-integer amounts (msPlayed/track length) and recreate the ‘by total plays’ analysis performed earlier. 

