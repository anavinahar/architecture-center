# Logging and monitoring



System metrics - CPU, memory, threads

Container metrics

Application metrics - latency, number of requests, number of errors

Application logging

Correlation IDs

Dependent service metrics

Structured logging

Client metrics


## Considerations

Configuration and management
    - Managed service vs deployed in the cluster

Ingestion rate
    - Will the data be downsampled automatically
    - Can you do sampling in the client?
    - Will you be throttled
    - What is the granularity of the data.

What can it log 
    - Prometheus only supports floats, not strings. 
    - 

Storage costs

Resiliency - where is the data stored, can it be lost if a node goes down?



## Technical options

Application Insights

    Throttles after 32k/sec. To avoid this, use sampling.
        - Throttling can lead to lost data. (Client SDKs may try to resend)
    Sampling - can be done in the server app (Adaptive sampling) or in AI (ingestion sampling)
        - Ingestion sampling reduces storage but not telemetry traffic
    - There is also a daily cap - this is only meant as a final protection against too much data

Prometheus - time series database 