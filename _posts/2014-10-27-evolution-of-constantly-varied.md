---
layout: post
category : [play]
tagline: ""
tags : [R, Python]
---
{% include JB/setup %}


Crossfit workouts (WODs) vary day to day and cover a broad spectrum of movements in order to help everyday athletes advance in every possible direction. In light of this, I examined how movements featured in WODs have evolved over time. Note, visualizations below capture number of WODs containing a given movement/named-WOD in a year. The last year captured is 2014 with 2 months left in the game.  WODs from 2001 onwards  reveal that:

- The following is well behind us

![Tabata!!!]({{lubagloukhov.github.com}}/assets/tabata.jpg )

![Movement Plot]({{lubagloukhov.github.com}}/assets/movePlot.jpeg )


- 'Cindy' is the Drew Barrymore of 'The Girls': SUPER POPULAR, residual popular, minor crash & burn, COMEBACK with steady & strong success

![Drew]({{lubagloukhov.github.com}}/assets/drew.jpg )
![Named WOD Plot]({{lubagloukhov.github.com}}/assets/namedPlot.jpeg )

- &, as I suspected, Swimming has been making a minor splash in recent years. "For three consecutive years, the CrossFit Games have had a workout that contained swimming in one format or another." [boxlifemag](http://www.boxlifemagazine.com/training/should-you-incorporate-swimming-in-your-crossfit-programming)

![Swim]({{lubagloukhov.github.com}}/assets/shark.jpg )
![RBSR Plot]({{lubagloukhov.github.com}}/assets/rbsrPlot.jpeg )

	
I scraped the WOD Archive (ex: [http://www.crossfit.com/mt-archive2/2014_10.html](http://www.crossfit.com/mt-archive2/2014_10.html)) using Python and visualized the text using R. In the process, I attempted to to do the full bake in Python using the ggplot module. Unfortunately, the module leaves much to be desired, especially for those used to the rich flexibility of R's ggplot2(). However, for the functionality that does exist the syntax should taste familiar.

The DataFrame (df_lng) shows how many WODs ('value') contain a given movement ('variable') in a month (starting date indicated by 'date').

~~~ python
plot = ggplot(df_lng, aes(x='date', y='value'))+\
	geom_bar(aes(x='date', weight = 'value', fill='steelblue'))+\
	ylab('Number of Weekly Workouts w/ Snatch') + xlab('Date')
~~~
        
To scrape & massage the data, I used Python's Feedparser, BeautifulSoup and Pandas. For example, given l= a url like [http://www.crossfit.com/mt-archive2/2014_10.html](http://www.crossfit.com/mt-archive2/2014_10.html) .

~~~ python    
CFfp = feedparser.parse(l)
html = CFfp['feed']['summary']
soup = bs4.BeautifulSoup(html)

s = soup.find("div", {"class" : re.compile("date")})
for i in range(len(soup.findAll("div", {"class" : re.compile("date")}))-1):
	s = s.findNext("div", {"class" : re.compile("date")})
	workout = s.findNext('p')
	date = (str(s.getText())).strip()
	workout = remove_html_markup(str(workout)).replace('\n', ' ')
	wodDict[date]=workout
	i+=1
~~~

For each date/WOD, whether or not a movement occurs was flagged. This was captures in a dictionary logging the date that each word (movement) appears for a given list of 113 movements (& named wods like Nasty Girls).

~~~ python    
d2 = dict((datetime.strptime(k, '%B %d, %Y'), v) for k, v in wodDict.items())
dateDict ={}
for move in Movements:
	for day in d2:
		if len(re.findall(move,d2[day].lower(),flags=re.I))>0:
			if dateDict.has_key(move):
				dateDict[move].append(day)
			else:
				dateDict[move] = []
				dateDict[move].append(day)
~~~

I then dumped this to Pandas DataFrame and then a csv:

~~~ python
dateDF = pd.DataFrame(dict([(k,pd.Series(v)) for k,v in dateDict.iteritems()]))
dateDF.to_csv('dateDF.csv', sep=',')
~~~

From there, I handed it over to R's reshape, ggplot2, ggthemes and RColorBrewer to visualize the results. ggthemes recently came out with a FiveThirtyEight theme which I was able to take full advantage of by updating ggthemes using devtools.


Next Steps:

Current scrape captures just the first WOD listed of the day, to improve this I'd like to also capture alternate workouts as well as suggested weights, time, reps and distances. I'd like to examine any recurring movement sequences over multiple days that may occur when WODs build up to a complex movement. Successful scraping of WODs opens the door to scraping of user/athlete comments which can be used to analyze individual performance and sentiment.