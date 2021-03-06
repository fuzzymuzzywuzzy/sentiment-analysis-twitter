# sentiment-analysis-twitter
This script has the following features:
* Searches tweets using Twitter search API 
* Supports location-specific search using Twitter's geo search API
* Passes tweets through the VADER sentiment analysis model
* Summarises positive, neutral and negative sentiment
* Generates tweets in a .csv file output for further analysis

### VADER Sentiment Analysis Model

VADER (Valence Aware Dictionary and sEntiment Reasoner) is a lexicon and rule-based sentiment analysis tool that is specifically attuned to sentiments expressed in social media. More details can be found in:

* [cjhutto](https://github.com/cjhutto/vaderSentiment) - VADER Sentiment GitHub Repository

## Prerequisites

Install Python and the packages that is required to run this script. If you use Anaconda, most of these packages should already be installed. You will likely need to install the vaderSentiment package. Do so via your preferred command prompt: 

```python
pip install vaderSentiment
```

## Script

```python
import pandas as pd
import tweepy
import time
import sys
from collections import OrderedDict
from pandas.io.json import json_normalize
```
The following are imported:  
* pandas - dataframe manipulation
* tweepy - Twitter APIs
* time - while loop
* sys - kill program
* OrderedDict - ordering JSON files
* json_normalize - normalizing JSON files
```python
consumer_key = ""
consumer_secret = ""
access_key = ""
access_secret = ""
    
#Twitter authorisation
def initialize():
    auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
    auth.set_access_token(access_key, access_secret)
    api = tweepy.API(auth, parser = tweepy.parsers.JSONParser())
    return api

api = initialize()
```
Standard Twitter authorisation. Create your own app and obtain your own keys via [Twitter apps](https://developer.twitter.com/en/apps).
```python
print('\nThis program gets tweets using Twitter public APIs and passes them through the VADER sentiment analysis model - a model geared towards social media sentiment analysis. Results are generated in a .csv file output for further analysis.')    
geo_enabled = input('Run geo-enabled search? (y/n): ').lower().strip()

while geo_enabled not in ('y','n'):
    print('Invalid input. Please enter y/n: ')
    geo_enabled = input().lower().strip()
     
if geo_enabled[0] == 'y':
    country = input('Enter your country: ')
    geocode = api.geo_search(query = country)
    geocode = OrderedDict(geocode)

    while geocode['result']['places'] == []:
        print('\n No results found. Please try again.')
        country = input('Enter your country: ')
        geocode = api.geo_search(query = country)
        geocode = OrderedDict(geocode)

    #api.geo_search returns coordinates in 'longitude, latitude' but api.search geocode parameter is defined by 'LATitude, LONGitude'
    latitude = float(geocode['result']['places'][0]['centroid'][1])
    longitude = float(geocode['result']['places'][0]['centroid'][0])
    print('The lat-long coordinates for the country are ', latitude, ' ', longitude)
    max_range = int(input('Input radius of search in kilometres: '))
    searchterm = input('Enter your search term: ')
    searchquery = searchterm + ' -filter:retweets -filter:media -filter:images -filter:links'
    data = api.search(q = searchquery, geocode = "%f,%f,%dkm" % (latitude, longitude, max_range), lang = 'en', count = 100, result_type = 'mixed')

else:
    searchterm = input('Enter your search term: ')
    searchquery = searchterm + ' -filter:retweets -filter:media -filter:images -filter:links'
    data = api.search(q = searchquery, lang = 'en', count = 100, result_type = 'mixed')

data = OrderedDict(data)
data_all = list(data.values())[0]
max_tweets = input('Enter number of requested tweets (recommended less than 1,000): ')
max_tweets = int(max_tweets)

if data_all == []:
    print('\n No results found. Program will terminate.')
    sys.exit()
```
The above accomplishes the following: 
* We first ask the user whether a location-specific search should be conducted.
* If yes, we dig through the response from Twitter's geo_search API to fetch the coordinates of the country. 
* We also prompt the user to input a radius of search from the coordinate point.
* We then prompt the user to enter the search term. Retweets, media, images and links are automatically filtered out.
* We then prompt the user for a requested amount of tweets to fetch. Twitter's standard search API fetch a maximum of 100 tweets per call and limits 180 calls per 15 minutes. This means a maximum amount of 18,000 tweets can be mined per 15 minutes. 

The API then attempts to fetch the first 100 searches. Exception handling is in place in the event of no tweets.  
```python
done = False

if len(data_all) < 100:
    done = True

while not done:
    last_id = data_all[-1]['id']
    data = api.search(q = searchquery, lang = 'en', count = 100, result_type = 'mixed', max_id = last_id)      
    data = OrderedDict(data)
    data_all_temp = list(data.values())[0]
    data_all += data_all_temp
    time.sleep(1)
    
    if data_all_temp == []:
        done = True
        
    if len(data_all_temp) < 100:
        done = True
        
    if len(data_all) >= max_tweets:
        done = True

df = pd.DataFrame.from_dict(json_normalize(data_all), orient='columns')
```
We use a while loop with a boolean to repeatedly call the API until the requested number of tweets is met. This method references the last tweet ID and appends the next 100 tweets to the list. Results are returned in a JSON file. JSON file is ordered and tweets are extracted to a separate list. The resulting list is converted into a dataframe for further manipulation.
```python
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
analyzer = SentimentIntensityAnalyzer()

def sentiment_score_compound(sentence):
    score = analyzer.polarity_scores(sentence)
    return score['compound']

def sentiment_score_pos(sentence):
    score = analyzer.polarity_scores(sentence)
    return score['pos']

def sentiment_score_neg(sentence):
    score = analyzer.polarity_scores(sentence)
    return score['neg']

def sentiment_score_neu(sentence):
    score = analyzer.polarity_scores(sentence)
    return score['neu']
```
Import VADER model and using its methods, define separate functions to return compound, positive, negative and neutral scores. 
```python
df2 = pd.DataFrame()
df2['id'] = df['id'].values
df2['created_at'] = df['created_at'].values
df2['user.screen_name'] = df['user.screen_name'].values
df2['place.country'] = df['place.country'].values
df2['tweet'] = df['text'].values
df2['vs_compound'] = df.apply(lambda row: sentiment_score_compound(row['text']), axis=1)
df2['vs_pos'] = df.apply(lambda row: sentiment_score_pos(row['text']), axis=1)
df2['vs_neg'] = df.apply(lambda row: sentiment_score_neg(row['text']), axis=1)
df2['vs_neu'] = df.apply(lambda row: sentiment_score_neu(row['text']), axis=1)
```
Output dataframe (df2) is defined importing desired columns from tweets dataframe (df). Sentiment scores columns are generated applying respective functions to the tweet.

```python
no_pos_tweets = [tweet for index, tweet in enumerate(df2['vs_compound']) if df2['vs_compound'][index] > 0]
no_neg_tweets = [tweet for index, tweet in enumerate(df2['vs_compound']) if df2['vs_compound'][index] < 0]
no_neu_tweets = [tweet for index, tweet in enumerate(df2['vs_compound']) if df2['vs_compound'][index] == 0]
```
Positive / negative / neutral tweets are counted as a % of total tweets fetched. 

```python
print(len(df2), 'tweets found.')
print('\rPercentage of positive tweets: {:.2f}%'.format(len(no_pos_tweets)*100/len(df2['vs_compound'])))
print('\rPercentage of negative tweets: {:.2f}%'.format(len(no_neg_tweets)*100/len(df2['vs_compound'])))
print('\rPercentage of neutral tweets: {:.2f}%'.format(len(no_neu_tweets)*100/len(df2['vs_compound'])))
```
High-level information of tweets fetched are reported.

```
header = ['id', 'created_at', 'user.screen_name', 'tweet', 'vs_compound', 'vs_pos', 'vs_neg', 'vs_neu']
timestr = time.strftime("%Y%m%d-%H%M%S")
df2.to_csv('output-'+timestr+'.csv', columns = header)

print(len(df2), 'tweets found.')
print('Output saved as output-',timestr,'.csv')
```
Final dataframe is output into a .csv file for further analysis. 

### Limitations
1. Twitter standard search API have a maximum look back window of 7 days. 
2. More than 90% of tweets have geocodes disabled, so searching for location-specific tweets may yield less meaningful results. 
3. Experimented using regex to clean up tweets. Before-after impact was minimal (~5% of original sentiment results affected).  
