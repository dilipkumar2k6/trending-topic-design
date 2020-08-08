# trending-topic-design
- Hash tag in tweet is consider as topic
- scale of tweeter 
- interval? trending topic of last 10 seconds
# Simple approach
- Client requests for trending topics
- Server fetches the latest tweet in the last ten seconds from the database and return the top K hashtags as trending topics
Issues?
- Not scalable
- Query database is expensive as same data is queried again and again
# Improved design
- Add load balancer
- multiple servers handling request
- Master slave architecture, and read from slave database
Issue?
- Every 10 seconds, you end up query same tags
Two approach
- Sliding window
- Caching
# Sliding window
- maintain a hash table with topic and tweet count over the last 10 seconds
- Every seconds, update the dictionary
    - Add new topic tweet count over the last seconds from the database
    - Remove tweet that are no longer in 10 seconds window
- Return topmost hashtags in the hash table

How big the hash table?
- 1 million users per seconds
- each one send 10 tweets
- total 10 tweets per seconds
- each tweet of size 5 bytes
- 50 MB per seconds
- 500MB per 10 seconds

# Reference
https://www.youtube.com/watch?v=RMuLavkPLwc
