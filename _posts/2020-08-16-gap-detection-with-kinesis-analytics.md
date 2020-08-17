---
layout: post
title:  "Timeseries Gap Detection with Kinesis Analytics"
date:   2020-08-16 19:00:00 -0400
categories: aws kinesis-analytics
---

At Enel X we have thousands of meters at commercial and industrail sites
tracking power usage. We use this meter data to measure power consumption
and the ability for a given site to reduce load on the electrical grid at
critical moments to ensure overall grid stability.

But many of these meters are poorly connected – over cell modem connections
in places of lots of interference, or old hardware, or unreliable data sources.
So we often need to be able to detect when gaps occur in the incoming timeseries
data stream.

For our data ingestion pipeline, we've hooked up AWS Lambda functions to a
Kinesis stream for processing the intervals as they arrive. To find when a gap
has occured, we turn to
[Kinesis Analytics](https://aws.amazon.com/kinesis/data-analytics/).

Kinesis Analytics lets your run SQL commands on continuous streams of JSON
payloads. The trick to gap detection is to LEFT JOIN the stream to itself
on the interval's preceeding timestamp. We use a Lambda preprocessor function
to decorate each interval with these attributes.

```
CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM" (
    stream_id VARCHAR(64),
    closure_ts BIGINT,
    received_dttm TIMESTAMP
);

CREATE OR REPLACE PUMP "STREAM_PUMP" AS INSERT INTO "DEDUPE_STREAM"
SELECT STREAM
    window1."stream_id",
    window1."interval_ts" AS "closure_ts"
    ROWTIME AS "received_dttm"
FROM  "INCOMING_001" OVER (RANGE INTERVAL '60' SECOND PRECEDING) AS window1
LEFT JOIN  "INCOMING_001" OVER (RANGE INTERVAL '6' HOUR PRECEDING) AS window2
    ON
        window2."stream_id" = window1."stream_id" AND
        window2."interval_ts" = window1."prior_ts"
WHERE
    window2."interval_ts" IS NULL;
```

We take the window of the last 60 seconds, and LEFT JOIN to the window of the
last 6 hours. Any row that did not join with a preceeding interval is the
closure of a gap! So we found the incoming intervals that closed a timeseries
gap and we can send those closures out to our estimation system.
