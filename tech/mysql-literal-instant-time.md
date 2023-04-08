# Literal vs Instant Time Value

Generally, there are two types of format for time in the Java world: literal and instant. Sometimes it's puzzling regarding the difference between these two and when to use literal or instant in our software system and DB storage. 

## What is a literal?
Literal represents a date and a time-of-day, without a time zone. It remains the same for users in different time zones. Like Christmas, it's always December 25 no matter where you are. With that said, literal doesn't represent a moment, but only [a potential moments along a range of about 26-27 hours, the range of all time zones around the globe](https://stackoverflow.com/questions/32437550/whats-the-difference-between-instant-and-localdatetime). Some examples of literals are "2023-01-01", "2023-01-01 09:00:02". 


## What is an instant?
An Instant, or [Epoch](https://currentmillis.com/), is a moment on the timeline in UTC,  the number of nanoseconds that have elapsed since 1970-01-01 00:00:00 in UTC. It represents a unambiguous, concrete moment. It can have different literal values in different time zone. For example, Instant 1680962401065 can be converted to "2023-04-08 22:00:01 CST", or "2023-04-08 14:00:01 UTC".


## How does MySQL save time?
MySQL saves `DATE` and `DATETIME` types in literal format. Data are encoded into 5 bytes, with no time zone info. `TIMESTAMP`, however, is saved with Epoch value, 4 bytes when no fraction. MySQL converts `TIMESTAMP` values from the current time zone to UTC for storage, and converts it to current time zone for retrieval. One thing to note is that we need to make sure the same time zone is used for conversion in both directions.  Otherwise, the value retrieved will be different from what's stored.


## Reference: 
https://dev.mysql.com/doc/refman/8.0/en/datetime.html