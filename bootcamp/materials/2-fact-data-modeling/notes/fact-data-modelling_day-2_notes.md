## Is it a fact or a dimension?

- Did a user log in today?
    - The log in event would be a fact that informs the "dim_is_active" dimension
    - VS the state "dim_is_activated" which is something that is state-driven, not activity driven
- You can aggregate facts and turn them into dimensions
    -Is this person "high engager"? A "low engager"?
        - Think `scoring_class` from week 1
    - CASE WHEN to bucketize aggregated facts can be very useful to reduce the cardinality

## Properties of Facts vs dimensions

- Dimensions
    - Usually show up in GROUP BY when doing analytics
    - Can be "high cardinality" or "low cardinality" depending
    - Generally come from a snapshot of state

- Facts
    - Usually aggregated when doing analytics by things like SUM, AVG, COUNT
    - Almost always higher volume than dimensions, although some fact sources are low-volume, think "rare events"
    - Generally come from events and logs

## Airbnb example

- Here's a crazy blurry example:
    - Is the price of a night on Airbnba fact or a dimension?
    - The host can set the proce which sounds like an event
    - It can easily be SUM, AVG, COUNT'd like regular facts
    - Prices pn Airbnb are doubles, therefore extremely high cardinality
        - Prices can be considered as dimension though as it is an attributes of one night
    - The fact in this case would be the host changing the setting that impacted the price (Price discount). When they change the settings, that would log an event, which is a fact.
    - Think as a fact has to be logged, a dimension comes from the state of things
    - Price being derived from settings is a **dimension**. Feels like a fact but actually a dimension

## Boolean/ Existence-based Fact/ Dimensions

- dim_is_active, dim_bought_something, etc
    - These are usually on the daily/hour grain too
- dim_has_ever_booked, dim_ever_active, dim_ever_labeled_fake
    - These "ever" dimensions look to see if there has "ever" been a log and once it flips one way, it never goes back
    - Interesting, simple and powerful features for machine learning
        - An Airbnb host with active listings who has never been booked
            - Looks sketchier and sketchier over time
- "Days since" dimensions (e.g. days_since_last_active, days_since_signup, etc)
    - Very common in Retention analytical patterns
    - Look up J curves for more details on this

## Categorical Fact/ Dimensions

- Scoring class in Week 1
    - A dimension that is derived from fact data
- Often calculated with CASE WHEN logic and "bucketizing"
    - Example: Airbnb Superhost

## Should you use dimensions or facts to analyze users?

- Is the "dim_is_activated" state or "dim_is_active" logs a better metric?
    - Great data science question because it depends!
- It's the difference between "signups" and "growth" in some perspectives

## The extremely efficient Date List data structure

- Go and read Max Sung's writeup on this: Link
- Extremely efficient way to manage user growth
    - The annoying thing is: Need to check if this user is monthly active then we have to look at the last 30 days of facts and GROUP BY, eventhough 29 of the 30 days don't change, need to check the data everyday when at least only 1 day the user is active
- Imagine a cumulated schema like 'users_cumulated'
    - User_id
    - Date - current date
    - Dates_active - an array of all the recent days that a user was active
- You can turn that into a structure like this
    - User_id, date, datelist_int
    - 32, 2023-01-01, 100000010000001 (every bit is one day back, powerful to store this data as integer type because we get very good compression)
    - Where the 1s in the integer represent the activity for 2023-01-01 - bit_position (zero indexed)

