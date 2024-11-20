## Idempotent pipelines are CRITICAL

- Your pipeline produces the same results regardless of when it's ran
- What does idempotent mean?
    - denoting an element of set which is unchanged in value when multiplied or otherwise operated on by itself

## Pipelines should produce the same results

- Regardless of the day you run it
- Regardless of how many times you run it
- Regardless of the hour that you run it

## Why is troubleshooting non-idempotent pipelines hard?

- Silent failure! (It actually doesn't fail, but produces different or incorrect data or non-reproducible data)
- You only see it when you get data inconsistencies and a data analyst yells at you

## What can make a pipeline not idempotent

- INSERT INTO without TRUNCATE (INSERT INTO without putting TRUNCATE will duplicate the data)
    - It's better to use MERGE or INSERT OVERWRITE every time
    - MERGE will not duplicate the data if new and old data matches
- Using start_date > without a corresponding end_date <
    - E.g. WHERE date is greater than yesterday, if we run it today we will get 1 day of data, if we run it tomorrow we will get 2 days of data.
    - Putting end_date ensure we have that window of data
- Not using a full set of partition sensors
    - Pipeline might run when there is no/ partial data
- Not using depends_on_past for cumulative pipelines
    - Another term is sequential processing, when pipeline is depending on yesterday's data (pipeline is not running in parallel)
- Relying on the "latest" partition of a not properly modeled SCD table
    - Zach has so much pain at Facebook, DAILY DIMENSIONS and "latest" partition is a very bad idea
    - Cumulative table design AMPLIFIES this bug
- Relying on the "latest" partition of anything else

## The pains of not having idempotent pipelines

- Backfilling causes inconsistencies between the old and restated data
- Very hard to troubleshoot bugs (unit test will still pass if your pipeline is idempotent)
- Unit testing cannot replicate the production behaviour.
- Silent failures

## Should you model as Slowly Changing Dimensions?

- SCD is a dimension that changes overtime e.g. age, android user change to iphone
- Max, the creator of Airflow HATES SCD data modelling
- What are the options here?
    - Latest snapshot - instead of modelling day by day, you have whatever the current value is and that's what you ues. The problem is if you have a SCD and you only hold on to the latest value that means the pipeline is idempotent because if you backfill the data then the data will pickup the new SCD value instead of the old one e.g. FB post from 18 years old turns out to be like it was posted when Zach is 29 years old(current age)
    - Daily/Monthly/Yearly snapshot - when backfilling it picks up the value of SCD because we have daily snapshot
    - SCD - instead of having 365 rows for the same attributes let's just have one rows that says when the attribute change. If the dimension is really slowly changing then we will get the benefit of compression. If it is rapidly changing (once a week) then it is better to do daily snapshot

## Why do dimensions change

- Someone decides they hate iPhone and want Android now
- Someone migrates from USA to another country

## How can you model dimensions that change?

- Singular snapshots
    - BE CAREFUL SINCE THESE ARE NOT IDEMPOTENT
- Daily partitioned snapshots
- SCD Types 1, 2, 3

## The types of Slowly Changing Dimensions

- Type 0
    - Aren't actually slowly changing (e.g. birth date)
- Type 1
    - You only care about the latest values
    - NEVER USE THIS TYPE BECAUSE IT MAKES YOUR PIPELINES NOT IDEMPOTENT ANYMORE
    - For OLTP, using online apps it is fine to use latest/current value because you don't really need to look at the past
    - As data engineers who cares for analytics, don't use this
- Type 2
    - You care about what the value was from "start_date" to "end_date"
    - Current values have either and end_date that is NULL or far into the future like 9999-12-31
    - Hard to use:
        - Since there's more than 1 row per dimension, you need to be careful about filtering on the time
    - The only type of SCD that is pure IDEMPOTENT
- Type 3
    - You only care about "original" and "current"
    - Benefit is you only have 1 row per dimension
    - Drawback is you lose the history in between original and current is it changes more than once
    - It is partially idempotent which means it is not

## Which types are idempotent?

- Type 0 and Type 2 are idempotent
    - Type 0 is because the values are unchanging
    - Type 2 is but you need to be careful with how you use the start_date and end_date syntax
- Type 1 isn't idempotent
    -If you backfill with this dataset, you'll get the dimension as it is now, not as it was then!!
- Type 3 isn't idempotent
    - If you backfill with this dataset, it's impossible to know when to pick "original" vs "current"

## SCD2 Loading

Two ways:

- Load the entire history in one query
    - Inefficient but nimble
    - 1 query and you're done
- Incrementally load the data after thr previous SCD is generated
    - Has the same "depends_on_past" constraint
    - Efficient but cumbersome