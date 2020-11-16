# trending-topic-design
- What is topic?
    - Hash tag in tweet is consider as topic
- scale of tweeter 
    - 1 million users per hour
- interval? 
    - top 20 trending topic of last 10 minutes in 1 hour range
# Simple approach
- Client requests for trending topics
- Server fetches the latest tweet in the last one hour from the database and return the top K hashtags as trending topics

Issues?
- Not scalable
- Query database is expensive as same data is queried again and again

# Sliding window Approach 
## Sliding Window
- Top trending topic means the topics which have maximum occurrence in tweets in last 60 minutes.
- To model the time period, system uses the notion of “Window”.
- Window is a logical unit which monitors topic for specified time period.
- So Window of 1 hour would keep track of topic for 1 hour time period. 
- To keep track of different topics, we would need one window per topic. 
- Window is sub-divided into equal time period chunks called as “pane”.
- So, if we divide the Window into 6 equal chunks then each pane would be of 10 minutes. 
- Below figure, shows window of 1 hour and 6 panes of 10 minutes each, for topic, say, #hangover.
![](http://3.bp.blogspot.com/-qBQh3eA8ayM/Ua_-7geO8lI/AAAAAAAABqI/jaGfJ2j0fS8/s1600/image005.gif)
- Each pane of the window would keep track of following in the pane time period
    - total occurrence of the topic (count) 
    - positive sentiment score
    - negative sentiment score
- “Green Check” in the above figure means that the “pane” is a valid pane.
- For monitoring topic activity (count, sentiment score) for last one hour, at every 10 minutes, the oldest pane (leftmost) would expire (making pane invalid), and a new pane would be added at rightmost end.
- This mechanism is called as “Window Slide”. 
- Before invalidating the oldest pane, an aggregation operation is performed on complete window which finds following to gives the summary about the topic during last one hour.
    - total count
    - total positive sentiment score
    - total negative sentiment score 

    ![](http://3.bp.blogspot.com/-hDe2ksEnvOU/Ua__AOFTphI/AAAAAAAABqQ/2_6es6oLG5s/s1600/image006.gif)
- For implementing the “Window” concept, Circular Linked List data structure is used. Below figure, shows how the window can be modeled as Circular Linked List.

![](http://1.bp.blogspot.com/-W9juuzj-ID4/Ua__HIgDmaI/AAAAAAAABqY/MuMMC8iiJnE/s1600/image007.gif)

![](http://3.bp.blogspot.com/-6vuXJzDDl1E/Ua__MBLkrNI/AAAAAAAABqg/XwF-oIYKJ2U/s1600/image008.gif)
- Elegant thing about the representation of the Window as a Circular Linked List is that circular linked list implementation only need to expose two APIs:
```
//API to add data to the latest pane
Void addData(Object data);
//API to return aggregate information and also do sliding at end of aggregation
Object getAggregateInfoAndSlide();
```
- Thus, the client which consumes this, does not need to know how many “panes” or “nodes” are present during add, slide, aggregate operation. 
- New panes can be added without client dependency. 
- Even without knowing underline window/pane implementation, client can still control the length of the pane (pane length would be equal to time difference between two consecutive calls to “getAggregateInfoAndSlide()” function). 
- This makes integration of circular link list data structure very easy, useful and simple.
- Window mechanism described only tracks activity for one topic. 
- But, for finding top topics it is required to monitor activity for all the topics which are encountered in last one hour time period. 
- For tracking activity for all the topics, a Map data structure is used, where key is the topic name and value is the pointer to the “Window” or the Circular Linked List. Below figure, shows the data structure.

![](http://1.bp.blogspot.com/-XpauA8orH4c/Ua__SbOHeKI/AAAAAAAABqo/pP6GlbkKQqs/s1600/image009.gif)
- Approach for finding Top Topics from the Twitter Stream

![](http://3.bp.blogspot.com/-4yxboi74zK8/Ua__YPUY52I/AAAAAAAABqw/Vgb_Une2ITU/s1600/image010.gif)

## Buffer Incoming Data
- This module is responsible for integrating with Twitter Streaming API via subscribing to the twitter stream. 
- twitter4j library is used for the subscription. 
- Once subscribed, it would receive live tweets from Twitter, and the modules buffers the incoming tweets, retrieves the tweet message, and then pass the tweet to the next module (Filter Content) for the processing of the tweets. 
- Following shows the buffered tweets and the output from the module

![](http://2.bp.blogspot.com/-w29jrQBp61g/Ua__eGq1GbI/AAAAAAAABq4/1IDbQdVBU8g/s1600/image011.gif)
## Filter Content
- This module receives a tweet message and is responsible for extracting relevant information from the tweet message and passing the relevant information to the next phase. 
- Relevant information extracted from the tweet:
    - Topic Names: Extracts all the hashtags that are present in the tweet
    - Positive Sentiment Score: Send the tweet message to the “Sentiment Calculator” and if the sentiment score is positive then stores the score as positive sentiment score
    - Negative Sentiment Score: Send the tweet message to the “Sentiment Calculator” and if the sentiment score is negative then store the score as negative sentiment score
- For each topic retrieved, pass the topic name, positive and negative sentiment score to the next module. 
- Following figure describes the module:

![](http://1.bp.blogspot.com/-xyx-XIswVGs/Ua__he1yhuI/AAAAAAAABrA/98mYrqbZTUs/s1600/image012.gif)
## Sentiment Calculator
- This module receives tweet message and it calculates the sentiment score for the message and returns the sentiment score to the caller. 
- For calculating the sentiment score, SentiWordNet dictionary is used. 
## Update HashTag Summary
- This module maintains the topic summaries for all the topics seen in the Twitter Stream. 
- The data structure used for maintaining the topic summaries is the same as that is shown in the previous section. 
- The module receives an object (topic name, positive sentiment score, and negative sentiment score) as input. 
- It then does a lookup in its Map to find the window/circular link list, for the topic name. 
- If no record found then a new entry is added to the map with the topic name and the object is added to the latest pane. 
- If the record exists then the object is added to the latest pane.

![](http://2.bp.blogspot.com/-FE22tmy18ho/Ua__lWH4SnI/AAAAAAAABrI/cMo8WI6NTdo/s1600/image013.gif)
## Slide and Send Window Summary
- This is not a separate module, but a logical different part of “Update HashTag Summary” module. 
- After fixed specified time, 10 minutes, on every element present in the map, which is maintained by “Update HashTag Summary”, `getAggregateInfoAndSlide()` function is called. 
- This method returns aggregated information (total count, total positive and negative sentiment score) for every topic for complete window. 
- This information (object) is passed to the next phase which finds top 20 topics (based on total count) from all the summaries.
## Find Top 20 HashTags
- This module receives topic summaries and every 10 minutes, it persist top20 topics that have maximum count, in the database with current timestamp.
## Overall topology
![](http://2.bp.blogspot.com/-SsbvcnYe-Ww/Ua__rXLK9AI/AAAAAAAABrQ/GdowQ46Esj4/s1600/image014.jpg)
# Rank Topics
- “Rank Topics” module ranks the topics on the basis of the notion of “trending”. 
- The topics which are relatively more aggressively trending are given higher rank. 
- In order to implement this module, a precise definition of “trending” is required. 
- In the project we shall consider a topic as a trending topic if it is among top k maximum occurred topics in last one hour (window time).
- This is a baseline criterion for a topic to be considered as a trending topic. 
- In order to rank the top k topics, intuitively a topic which is “growing fast” in the latest panes of the window should be ranked higher than the topics which grew in the older panes of the window and are dying off in the latest window panes. 
## Pure Count Based Ranking
- One of the naïve, easy to implement, strategy would be to rank the topics based upon the count of occurrence in the “Window” time period.
- It turned out that, this strategy has a major drawback. 
- Following two graphs are for two topics, and with the current strategy, topic in fist figure would be ranked higher than the topic in second figure, which is somewhat against our intuition.
![](http://2.bp.blogspot.com/-bHAN-RYsdGQ/Ua__0-V7T0I/AAAAAAAABrg/KP9_CN2uA10/s1600/image016.gif)
![](http://4.bp.blogspot.com/-nw7FUSaYgFk/Ua__5dHdRDI/AAAAAAAABro/mgmMzenQLA4/s1600/image017.gif)
## Local-Z Score Based Ranking
![](http://3.bp.blogspot.com/-At_jVwO7uJk/UbAAkapnkRI/AAAAAAAABr4/76wcSVweRl8/s1600/image019.gif)
![](http://1.bp.blogspot.com/-MFGRf7b6-_0/UbAAncEvpSI/AAAAAAAABsA/wLIjJ1xdK5E/s1600/image020.gif)
# Size estimate
## How big the in memory hash table?
- 1 million users per hour
    - each one send 1 tweets per 1 hour
    - total 1 millions tweets per 1 hour
- 20% of total tweets are equal to number of topics
    - Total number of topics: 200k 
    - Data per pane i.e. 10 minutes
        ```
        count: xxx int 4 bytes
        pSentScore: yyy double  8 bytes
        nSentScore: zzz double  8 bytes
        ```
    - Total size per pane: 20 bytes
    - Total size for per window: 120 bytes
    - Total size of 200k topics window: 200k * 120 bytes ~ 24 mb
    - Size of hash : 200k * 12 bytes = 2.4 mb
    - Total size: 26.4 mb 
## Database storage estimate
- Following is schema of data stored in database in every 10 minutes
```
{
    timestamp: datetime 8 bytes
    trends: ['#topic1', '#topic2',.....,'#topic20'] 20*120 bytes
}
```
- Size of each document: 2*4 bytes + 8 bytes + 2400 bytes ~ 2500bytes ~ 2.5kb
- In an hour: 15kb

# Reference
https://www.youtube.com/watch?v=RMuLavkPLwc

http://sayrohan.blogspot.com/2013/06/finding-trending-topics-and-trending.html


