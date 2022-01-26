# Portland SLR Data Analysis

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />
    
Analysis of sea level rise data from Portland Harbor, Portland Maine

This repository contains simplified versions of code used to analyze sea level 
rise trends, based on data from the Portland NOAA Tide Station.  We focus on 
four questions:

1.  Is sea level rising in Portland, and if so, how fast?  
2.  Is the rate of sea level change itself changing?  In particular, is 
    there evidence from the Portland Data that sea level rise is accelerating?  
3.  Has the frequency of tidal flooding changed over the past century?
4.  How do we expect the frequency of tidal flooding to change under moderate
    sea level rise scenarios?
    
Many of the analytic approaches we first explored when developing the State of
Casco Bay report have since been incorporated into an R package.  The package
facilitates conducting similar analysis for other NOAA tide stations. The
development version of the package is available on
[github](https://github.com/ccb60/SLRSIM).

# Statement of Purpose
CBEP is committed to the ideal of open science.  Our State of the Bay data
archives ensure the science underlying the 2020 State of the Bay report is
documented and reproducible by others. The purpose of these archives is to
release raw data and data analysis code whenever possible to allow others to
review, critique, learn from, and build upon CBEP science.

# Archive Organization
All CBEP 2020 State of the Bay data analysis repositories are organized into two
to four four sub-folders.  (One or more folders may be absent, if they were not
needed). Each subfolder contains data, code and analysis results as follows:

- Data. Contains data in simplified or derived form as used in our data analysis.
Associated metadata is contained in related Markdown documents, usually DATA_SOURCES.md and DATA_NOTES.md.

- Analysis.  Contains one or more R Notebooks proceeding through the data
analysis steps.  

- Graphics.  Contains R Notebooks stepping through development of related
graphics, and also raw copies of resulting graphics, usually in \*.png and
\*.pdf formats.  Because of downstream graphic design processes, graphics
included here may differ from how they appear in the State of the Bay.  

In this case, our goal was principally to recreate the graphic that NOAA makes
available to depict long-term sea level rise trends.  We conducted limited 
additional data analysis in response to comments by reviewers. That
supplementary analysis included:  
1.  A deeper dive into whether sea level rise in our region shows signs of
    accelerating, as has been widely predicted.
2.  Development of a graphic examining whether frequency of
    extreme tidal flooding has increased.

# Summary of Data Sources
The data used to produce the SLR graphic was downloaded directly from NOAA
Tides and Currents here:
[Portland, Maine SLR Info from NOAA](https://tidesandcurrents.noaa.gov/sltrends/sltrends_station.shtml?id=8418150).

The data description on the source web site says the following: 
> "[The data] shows the monthly mean sea level without the regular seasonal
  fluctuations due to coastal ocean temperatures, salinities, winds, atmospheric
  pressures, and ocean currents. ... The [data] values are relative to the most
  recent Mean Sea Level datum established by CO-OPS."

In other words, these data are not raw data, but have been pre-processed by
NOAA to remove seasonal patterns. The webpage declares the average Sea Level
Rise to be 1.89+/- 0.14 mm/yr, which is equivalent to a change of 0.62 feet in
100 years. We confirmed that result in our own analyses of these data.
   
Related observational monthly data, was downloaded via a NOAA API using a
simple python script. Data from the two data sources is highly correlated, 
but not identical, which presumably reflects NOAA's pre-processing of the data.

Additional hourly data was downloaded via the NOAA API to study probability of
daily tidal elevations exceeding a flood level equivalent to the current
"Highest Astronomical Tide" level for Portland, at 11.95 ft MLLW.

