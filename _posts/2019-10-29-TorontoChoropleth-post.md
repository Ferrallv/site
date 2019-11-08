---
title: "Using Census Data In A Choropleth"
excerpt_separator: "<!--more-->"
categories:
  - Project
excerpt: "A Toronto Choropleth using Bokeh"
---

# The Finished Product

<style>
#wrapper { width: 710px; height: 500px; padding: 0; overflow: hidden; }
#scaled-frame { width: 1000px; height: 2000px; border: 0px; }
#scaled-frame {
    zoom: 0.71;
    -moz-transform: scale(0.71);
    -moz-transform-origin: 0 0;
    -o-transform: scale(0.71);
    -o-transform-origin: 0 0;
    -webkit-transform: scale(0.71);
    -webkit-transform-origin: 0 0;
}

@media screen and (-webkit-min-device-pixel-ratio:0) {
 #scaled-frame  { zoom: 1;  }
}
</style>

<div id="wrapper"><iframe id="scaled-frame" src="https://torontochoropleth.herokuapp.com/2016TorontoChoropleth"></iframe></div>

# Why?

Some classmates and I were brainstorming ideas for data presentations that might be useful for immigrants thinking of moving to Toronto. From my experience living abroad, I felt that a visualization that informs where people of similar backgrounds are residing would be helpful. Moving to a new place can be stressful and isolating, and seeing locations where I might be able to find familiar foods and languages can be reassuring. The other aspects of the group project fell through but I decided to continue a test some of my skills. 

# How I got here, project-wise

To see the in-depth details of my data acquisition and transformation you can see this [Jupyter Notebook](https://github.com/Ferrallv/TorontoChoropleth/blob/master/TorontoChoropleth.ipynb). I used data from the 2016 Canadian Census and aggregated it into the Toronto neighbourhoods. In the future, I plan to add the data of the census from other years. 

To be able to share the interactive map I used Heroku to deploy my [Bokeh application](https://github.com/Ferrallv/TorontoChoropleth/tree/master/TorontoChoropleth).

The process broke down into three different phases: Data acquisition and transformation, plotting with bokeh and application design, and application deployment.

## Data Acquisition and Transformation

The data available from Stats Canada is already wonderfully clean and accessible. The only challenge was to extract the relevant immigration data for Toronto specifically. A step by step explanation is given in the previously mentioned jupyter notebook. The key to extracting the data was provided by the City of Toronto, who on request provided which census tracts (small geographic locations) applied to which neighbourhoods. 

## Plotting with Bokeh and Application Design

This was a wonderful learning experience with Bokeh and creating interactive plots. I had yet to write a python application to be used online and found this challenge exciting and enlightening. The app can be found in the repository mentioned earlier. The code has been commented to describe my processes and reasoning.

## Application Deployment

Heroku was a free option that I used to deploy the Bokeh application. It offers an excellent walkthrough for beginners to deploy their first app. For deploying a Bokeh app I struggled with writing a PROCFILE that was successful and so I'll post what worked for me here for others who might be having a likewise struggle.

`web: bokeh serve --port=$PORT --address=0.0.0.0 --allow-websocket-origin=yourapp.herokuapp.com --use-xheaders yourapp.py`


### Inspiration

[Plotting Choropleths from Shapefiles in R with ggmap – Toronto Neighbourhoods by Population](https://everydayanalytics.ca/2016/03/plotting-choropleths-from-shapefiles-in-r-with-ggmap-toronto-neighbourhoods-by-population.html)

### References

Statistics Canada. No date. CANSIM (98-401-X2016043). Last updated August 25, 2017.
https://www12.statcan.gc.ca/census-recensement/2016/dp-pd/prof/details/download-telecharger/comp/page_dl-tc.cfm?Lang=E (accessed October 29, 2019).

Social Development, Finance & Administration. Neighbourhoods. Last refreshed October 28, 2019.
https://open.toronto.ca/dataset/neighbourhoods/ (accessed October 29, 2019).

### Helpful pages

[A Complete Guide to an Interactive Geographical Map using Python](https://towardsdatascience.com/a-complete-guide-to-an-interactive-geographical-map-using-python-f4c5197e23e0)

[How to Scale iFrame Content in IE, Chrome, Firefox, and Safari](https://collaboration133.com/how-to-scale-iframe-content-in-ie-chrome-firefox-and-safari/2717/)
