## Components

- Poller that invokes the btc api and hydrates the dynamo table with the bitcoin prices
    - The poller writes the bitcoin prices to ddb with a conditional update
        - It avoids a full table scan during the write.
    - montecarlo/producer/lambda_function.py

- API Handler that serves requests
    - montecarlo/lambda/api_request_handler.py
        - Metrics:
            - For gathering all metrics, the metric request handler does a full table scan
                - This could be optimized looking up the metric_id key
            - For getting a specific coin's id, the request handler only looks up a specific key
              and not a table scan. This is accomplished using a filter expression on
              the coin_id
            - For ranking the metrics, currently we do a full table scan and get all data.
                - We could store the daily average as a separate column to optimize the amount of data held in memory

- Metric Gatherer with business logic to obtain the information requested by the api.
    - montecarlo/lambda/metric_gatherer.py
- Alert generation:
    - Alert generation relies on the alert string being written into the logs.
    - Email can then be decoupled from the poller this way.
        - We can use metric filters to search for and match terms, phrases, or values in your log events. When a metric filter finds one of the terms, phrases, or values in your log events, you can increment the value of a CloudWatch metric. 
        - This metric filter can then be used to trigger a lambda that generates emails.
        - This is a fair bit of work and possibly out of scope of this POC.

- Spins up a cdk stack with the data in us-east-2
    - [montecarlo_stack py](montecarlo_stack.py)

## Scalability

- Writing to and retrieving the daily data in Redis(ElasticCache) instead of looking it up in Dynamodb could enable faster lookups
    - Retrieval times would be faster for daily metrics
- Flushing the cache to dynamodb for data older than a day for longer term metric calculation would work better.
- Concurrent lambda executions can get expensive. One of the many knobs of cost control we have are:
    - Move the Metric Gatherer to an EC2/ECS instance and use a multi-threaded C++/Golang/Java application to split
      up the network requests to the BTC API and write them to the cache.

## Performance Optimization
- A storage optimization can be obtained by keeping a sliding window of daily metrics, by removing from the window
  and adding to the window every second, and also maintaining a running average with math.
Eg:
    - [MetricAge1,MetricAge2,......MetricAge24*60*60]
    - At Time 24*60*60 + 1:
        - Dequeue MetricAge1 from the Window.
        - Enqueue MetricAge24*60*60 into the Window

## Correctness:
Perhaps the biggest flaw with this implementation right now are:
- There are no alerts generated by this implementation. The most scalable way to generate alerts would be to:
    - Generate alerts based off a metric filter on the Cloudwatch logs
    - This metric filter would kick off a processing lambda that can send off custom alerts
- If the poller lambda fails, the metric aggregation will be buggy.
    - To obviate this, we definitely need a timestamp in the metrics pushed to dynamodb from the poller and use
      the timestamp to determine the daily window.
- Hardcoded coins since it was cumbersome getting all the coin pair combinations from a given index and testing
  them.

## Monitoring
- Metrics of importance:
    - API Gateway monitors on API 5xxes
        - Total API Invocation count
        - API Availability dashboards as a percentage of 1 - 5xxes/Total RequestCount
    - Lambda monitors:
        - Lambda invocation errors
        - Lambda invocation counts
        - Default lambda execution metrics
        - Alarms on Lambda DLQ
    - Dynamodb monitors:
        - Capacity monitors
        - RCU/WCU monitors

## Testing
- As the code stands, unit tests can be written for each component fairly easily using PyTest/unittest.
- Integration tests can be written using a long running canary that polls the APIs and checks whether they return
  the results periodically.
- Load testing can be simulated by setting up a test API endpoint and configuring the poller to interact with the
  fake API. This approach will be more useful when there are more than 3 coins as I've hardcoded them in the code.
- Manual testing detailed in API endpoints section

## Features implemented

- The app will query data from a publicly available source at least every 1 minute (try https://docs.cryptowat.ch/rest-api/ to get cryptocurrency quotes, 
- The app has a REST API to enable the following user experience (you do not need to implement the user interface):
  obtain metrics for a given id, and its rank.
- The app will log an alert whenever a metric exceeds 3x the value of its average in the last 1 hour. 

## API Endpoint to test with:
https://q74w4ov0i6.execute-api.us-east-2.amazonaws.com/prod

## API endpoints

    curl {API_ENDPOINT}/prod/metrics
        Gets available coins

    curl {API_ENDPOINT}/prod/metrics/dogeusd
    curl {API_ENDPOINT}/prod/rank/dogeusd
    curl {API_ENDPOINT}/prod/metrics/btcusd
    curl {API_ENDPOINT}/prod/rank/btcusd
    curl {API_ENDPOINT}/prod/metrics/ltcusd
    curl {API_ENDPOINT}/prod/rank/ltcusd

## Alerts

- Emits an alert of format: ("Alert " + coin_price + " " + coin_name) when price exceeds 3 * mean
- Future steps: 
    - Set up an SES email integration based off this log message: custom meric filter -> SES to be asynchronous while handling this alert.
    - The values in the db could be prefixed with the timestamp in case the hydrator for the values (poller
      component) crashes. Right now, we rely on those values being present and the poller never crashing.

### CDK Details

To manually create a virtualenv on MacOS and Linux:

```
$ python3 -m venv .venv
```

After the init process completes and the virtualenv is created, you can use the following
step to activate your virtualenv.

```
$ source .venv/bin/activate
```

If you are a Windows platform, you would activate the virtualenv like this:

```
% .venv\Scripts\activate.bat
```

Once the virtualenv is activated, you can install the required dependencies.

```
$ pip install -r requirements.txt
```

At this point you can now synthesize the CloudFormation template for this code.

```
$ cdk synth
```

To add additional dependencies, for example other CDK libraries, just add
them to your `setup.py` file and rerun the `pip install -r requirements.txt`
command.

```
source aws creds
```

```
cdk diff; cdk deploy
```
