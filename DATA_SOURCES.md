# Sources of Data
## Sea Level Rise in Portland, Maine

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />

**8418150_meantrend.csv** 
This data is pre-processed monthly water level data from NOAA, with certain
seasonal patterns removed. We use it principally to reprise and modify the sea
level rise graphic provided by NOAA on the Portland Tide Gage webpage.  The data
was downloaded directly from NOAA Tides and Currents Portland Tide Gage webpage,
using the "Export to CSV" button.
(https://tidesandcurrents.noaa.gov/sltrends/sltrends_station.shtml?id=8418150) 

Note:  The webpage declares the average sea level rise to be 1.89+/- 0.14 mm/yr
which is equivalent to a change of 0.62 feet in 100 years. 

**portland_tides_hourly.csv** 
This data file contains hourly tidal observations for the Portland Tidal Gage
going back to the beginning of the record, more than 100 years ago.  While more 
recent data is available on a six minute intervals, older data is available at 
hourly intervals, so we based most of our trend analysis on hourly data. 

We accessed the data directly from an on-line NOAA API using a simple python
script. Details on the API are available from the
[NOAA webpage](https://tidesandcurrents.noaa.gov/api/).

**portland_tides_hourly_predicts.csv**
This data file contains tidal predictions (astronomical tides) for the Portland
tide station, going back ~ 100 years.  Predictions are based on harmonic
constituents derived from observations during the current "tidal epoch", which 
ran from 1983 through 2001.  We calculated the difference between observed
and predicted tides, and used that data to support simulations of future water 
levels, and thus flood risk.

We accessed the data directly from the NOAA API using a simple python script.
Details on the API are available from the
[NOAA web page](https://tidesandcurrents.noaa.gov/api/).
