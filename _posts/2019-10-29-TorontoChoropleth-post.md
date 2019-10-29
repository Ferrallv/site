---
title: "Using Census Data In A Choropleth"
excerpt_separator: "<!--more-->"
categories:
  - Project
excerpt: "My friends and I were discussing ways to tell a story with data. I wanted to try my hand at choropleths, specifically using Bokeh and creating an interactive feature"
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

My friends and I were discussing ways to tell a story with data. I wanted to try my hand at choropleths, specifically using Bokeh and creating an interactive feature. I feel that data can create a stronger impact on the viewer when you can include them as a participant. So, I was thinking about Toronto neighbourhoods and I wondered if places like "Little Italy", "Little Portugal", or "Chinatown" had the highest population of immigrants who report their neighbourhoods namesake as their birthplace. 

# How I got here, project-wise

To see the in-depth details of my data acquisition and transformation you can see this [Jupyter Notebook](https://github.com/Ferrallv/TorontoChoropleth/blob/master/TorontoChoropleth.ipynb). I used data from the 2016 Canadian Census and aggregated it into the Toronto neighbourhoods. In the future I plan to add the data of the census from other years. 

To be able to share the interactive map I used Heroku to deploy my [Bokeh application](https://github.com/Ferrallv/TorontoChoropleth/tree/master/TorontoChoropleth).



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
