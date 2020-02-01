---
title: "Apartment Temperature - A little work on my cold living space"
excerpt_separator: "<!--more-->"
categories: 
  - Project
---



## An old building and no thermostat

I live in a place where it gets cold outside. The inside of my apartment gets cold too. The building controls the heat and we've done what we can to help warm our space up (plastic film insulation, blocking exhaust vents when not in use, uncovering the radiator, electric space heater, etc.). 

I'm non-confrontational, but this is a problem. So as anyone who is as risk-averse as I am would,  I decided to collect data so that I had proof of the fact it was too cold in my apartment. Then I thought, why stop there? Let's get something going to see if I can predict the temperature in my apartment given a set of variables!

I collected temperature data using [this temperature logger](https://www.elitechustore.com/products/elitech-rc-5-pdf-usb-temperature-data-logger-32000-points-reusable?_pos=4&_sid=472efe853&_ss=r) in a spot that would accurately record the "room" temperature. So it was suspended away from walls, about head height and near the middle of our space. 

I then aggregated weather data for the time I recorded as well as my electricity use habits with the hope that the space heater use would show up in the data. 

A full description of data collection and cleaning processes can be found in a notebook at this [repository](https://github.com/Ferrallv/ApartmentTemperature). 

The long and the short of it is, it's too cold.

{% include Apartment_Temperature_Over_Time.txt %}

In my data analysis I found that for this stretch of time it was too cold 55% of the time. Yikes! 

What else I found interesting was that, on average, I used electricity the most around dinner time and the late evening! Dinner makes sense, and my thoughts on the late evening was that heater was working hardest when it was coldest right before we turn it off to sleep.

We can take a look at the ordinal results of the average hourly recordings.

{% include Scaled_Data_for_Ordinal_Comparison.txt %}

This was all pretty straightforward. The hardest part was waiting a month to record a enough data points. Now it will take a year (!) before I have enough to run some meaningful regressions.

Leaving it at that felt too lacklustre to me. Touching and moving data always gives me a feeling of exploration and surprise so I slapped together a little app using Plotly Dash and Heroku to show some of the daily data recordings. 

<iframe src=https://apartment-dash-app.herokuapp.com/></iframe>
