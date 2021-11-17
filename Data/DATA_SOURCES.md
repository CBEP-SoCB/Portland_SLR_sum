# Sources of Data
## Sea Level Rise in Portland, Maine

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />

**8418150_meantrend.csv**  
Data downloaded directly from NOAA Tides and Currents (Using "Export to CSV"
button).
(https://tidesandcurrents.noaa.gov/sltrends/sltrends_station.shtml?id=8418150) 
June 6, 2020 by Curtis C. Bohlen

Note:  The webpage declares the average sea level rise to be 1.89+/- 0.14 mm/yr
which is equivalent to a change of 0.62 feet in 100 years. 

**portland_tides_hourly.csv**  
Data downloaded directly from NOAA API using a simple python script on
12/15/2020, by Curtis Bohlen.
Details on the API are available from the
[NOAA web page](https://tidesandcurrents.noaa.gov/api/).

This data was used to generate daily counts of exceedences above flood levels,
where flood levels are defined as observations above Portland's highest 
Astronomical Tide (HAT), at 11.95 MLLW.

**portland_tides_hourly_predicts.csv**  
Data downloaded directly from NOAA API using a simple python script on
12/15/2020, by Curtis Bohlen. Details on the API are available from the
[NOAA web page](https://tidesandcurrents.noaa.gov/api/).

This data was used with the preceding data to model future SLR based on time 
series models.


