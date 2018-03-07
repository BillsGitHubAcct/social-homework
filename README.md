### Twitter Bot
Searches on primary twitter account @Billbillwilson for @PlotBot Analyze: @somehandle,
When found the handle @somehandle is analyzed using TextBlob's sentiment analyzer.
Next a line chart is plotted and the chart is saved to a @somehandle.png in the local directory.
Finally a tweet with the plot png is tweeted to @PlotBot5 and the process goes to asleep
for 15 minutes before waking up again. 

```python
##### Twitter Bot #####
# Dependencies
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import json
import tweepy
import time
import seaborn as sns
import sys
from textblob import TextBlob

# Twitter API Keys
from config import (consumer_key, 
                    consumer_secret, 
                    access_token, 
                    access_token_secret)

# Setup Tweepy API Authentication
auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
api = tweepy.API(auth, parser=tweepy.parsers.JSONParser())

def get_mention(target_user, already_used):
    """
    Search through current account's most recent tweets and 
    parse each tweet for the string: "@PlotBot Analyze: @SomeTwitterHandle,"
    If it is found in the tweet and the handle "@someTwitterHandle" has not already been 
    analyzed, then stop and return it.  Otherwise continue reading and parsing the next tweet.
    
    """
    print('in get_mention')
    public_tweets = api.user_timeline(target_user, result_type="recent", tweet_mode='extended')
    mention_found = False
    men = ''
    for tweet in public_tweets:
        #tweet_stripped_words = TextBlob(tweet['full_text'])
        tweet_words = tweet['full_text'].split()
        try:
            found_idx = tweet_words.index("@PlotBot")
            if tweet_words[found_idx + 1] == 'Analyze:': 
                    men = tweet_words[found_idx + 2]
                    if men[-1] == ',':
                        men = men[:-1]
                    if men not in already_used:
                        print('Found good handle = %s' % men)
                        break
                    else:
                        print('Found handle =  %s' % (men))
                        print('Skip... already used')
                        men = ''
                        pass
            else:
                print('Skip Analyze: not found')
                pass
           
        except Exception as e:
            print('Mention not found: ' + str(e))
            pass
    return men
                        
def get_sentiment_values(mention):
    """
    Read up to 500 tweets for the passed in handle and load into dataframe
    Afterwards plot and save graph and png.
    
    """
    # Counter
    counter = -1
    # Variables for holding sentiments
    sentiments = []
    tweet_text = ''
    # Loop through 500 tweets
    for x in range(25):
    # Get all tweets from target user
        public_tweets = api.user_timeline(mention, page=x, tweet_mode="extended") 
        for tweet in reversed(public_tweets):
        # Run TextBlob Sentiment Analysis on each tweet
            TextBlob_sentiment = TextBlob(tweet['full_text'])
            
            # Add sentiments for each tweet into an array
            sentiments.append({"Date": tweet["created_at"], 
                               "Tweets Ago": counter,
                               "TextBlob" : TextBlob_sentiment.sentiment.polarity})
            # Add to counter 
            counter -= 1
        
    # Convert sentiments to DataFrame
    sentiments_pd = pd.DataFrame.from_dict(sentiments)
    #print(sentiments_pd)
    
    # Create plot
    # Plot for Sentiment Analysis
    fig, ax = plt.subplots()
    fig.suptitle(("Sentiment Analysis of Tweets (%s)" % (time.strftime("%x"))), fontsize=12)
    fig.tight_layout() # remove space between header and title
    fig.subplots_adjust(top=.93) # adjust so title is just above chart
    fig.set_size_inches(10, 6)
    #y = sentiments_pd["Compound"].tolist()
    y = sentiments_pd["TextBlob"].tolist()
    lower = -1 * (len(y)-1)
    x = np.arange(lower, 1, 1)
    ax.set_xlim(lower-7, 7, 25)
    ax.set_ylim(-1.05, 1.05,.1)
    
    # Shrink current axis by 10%
    box = ax.get_position()
    ax.set_position([box.x0, box.y0, box.width * 0.90, box.height])
    ax.plot(x,y, marker="o", linewidth=0.5, alpha=0.8, label=mention)
    
    ax.set_xlabel("Tweets Ago")
    ax.set_ylabel("Tweet Polarity")
    ax.xaxis.grid(color='white', linestyle='solid', linewidth=1)
    ax.yaxis.grid(color='white', linestyle='solid', linewidth=1)
    ax.set_axisbelow(True) # show plots on top of grid lines
    ax.set_facecolor('lightgray')
    lgd = ax.legend(title='Tweets', fontsize='small', mode = 'Expanded', numpoints = 1, scatterpoints = 1, loc= "upper left",
              bbox_to_anchor = (1, 1), labelspacing = 0.5)
    ax.tick_params(axis=u'both', which=u'both',length=0) # hide tick marks still show lables
    sns.despine(left=True, bottom=True, right=True) # remove border around chart     
    # Save the figure
    fig.savefig(mention, bbox_extra_artists=(lgd,), bbox_inches='tight')
    plt.savefig(mention)

    print('Plotted graph and saved graph for %s' % (mention))
                     
def tweet_out_graph():
    """
    Tweet out graph to @PlotBot5
    """
    api.update_with_media(mention + '.png',
                      'Sentiment Analysis Chart for @PlotBot5')
    print('Tweeted graph of %s to @PlotBot5' % (mention))  
```


```python
#################################################
# Main Driver
#################################################
used_mentions = []
while (True):
    print('Processing...')
    mention = ''
    target_user = '@Billbillwilson'
    mention = get_mention(target_user, used_mentions)
    if  mention != '':
        used_mentions.append(mention)
        get_sentiment_values(mention)
        tweet_out_graph()
    else:
        print('No new mentions found')
    print('Going to sleep for 5 minutes')
    for remaining in range(300, 0, -1):
        sys.stdout.write("\r")
        sys.stdout.write("{} seconds remaining.".format(remaining))
        sys.stdout.flush()
        time.sleep(1)
    sys.stdout.write("\rWaking up!            \n")
```
