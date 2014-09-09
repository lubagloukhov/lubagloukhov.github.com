---
layout: post
category : [play]
tagline: ""
tags : [R, Python]
---
{% include JB/setup %}


Do whisky flavor profiles differ by distillery region? part Two!

About a year ago, I had the pleasure of playing with a dataset of whisky flavor profiles. The data had latitude/longitude coordinates along with a numerical score for various flavor notes (sweet, medicinal, tobacco,...) of 83 whisky dstilleries.  The following resulted from a a geospatial visualization coupled with a k-means clustering of the distilleries based on flavor notes. 


![Whisky Kmeans]({{lubagloukhov.github.com}}/assets/kmeans.jpg )
[https://github.com/lubagloukhov/whiskies](https://github.com/lubagloukhov/whiskies)

In this post, I revisit the question of whether whisky flavor profiles differ by distillery region with an exploratory analysis of text reviews and data from [whiskybase.com](http://www.whiskybase.com/). I scraped the reviews of some 26,000 bottlings of whiskies and visualized the words used to describe whisky by region.

![WordClouds by Dsitillery]({{lubagloukhov.github.com}}/assets/byDist.jpg)
[https://github.com/lubagloukhov/whiskies2](https://github.com/lubagloukhov/whiskies2)

In scraping for reviews and background information,  I used Python (BeautifulSoup and urllib) to get the html code and extract text from within the html tags. I created a compilation of reviews, ratings and background in the form of a csv. Once I had the data & reviews in a csv, I used R (tm, openNLP and wordcloud) to visualize the most frequently occurring words in reviews of whiskys by distillery region (visualization shown), by age, by cask type, etc. tm was used to slightly clean & then split the data into words, counting the number of occurrences of each word. openNLP was used to identify nouns and adjectives, this helped to focus on descriptive words of the English language. 

The visualization indicates differentiation between whisky regions. Western Highlands appears to be full of spice, wood and raisins. Eastern Highlands have spice notes as well but more fruit and more sweetness. One of my favorite combinations is Islay single malt like Lagavulin with a chocolate covered, pretzel-topped caramel. The combination of rich caramel, deep chocolate and salty pretzel with sweet peat looks to be echoed by the Campbeltown region which I'm looking to explore next!

Next steps:

I'd love to see the word clouds by region overlaid on a map of Scotland. I wasn't successful in getting this off the ground with grid & wordcloud, perhaps a different set of tools is in order?

Some notes:

This was really my first independent project in Python. Since then, I've been formally studying data acquisition using Python. The code is a bit painful to look at (even for me). There are many things I would do differently, even now just 1 month later. I'd opt for dictionaries instead of dataframes and build in some error handling. [Whiskybase.com](http://www.whiskybase.com/) is not the cleanest source of data and, going back, I would like to spend more time tidying up my acquisition: ensuring no duplicate bottlings, ensuring only english reviews, filtering out non-English reviews, ...

