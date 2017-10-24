# Logging and monitoring

Monitoring to give you a holistic view of how the system is performing. Identify bottlenecks or services that are having problems. Services may be resource constrained, experiencing failures, or overwhelmed with requests.

Some issues that are common to MSA
- Network I/O limitations within the cluster
- Resource constraints due to running multiple containers on a single node, or containers with resource caps. 
- Backpressure. A service has a queue of requests it is processing, that causes upstream services to back up.
- Latency caused cascading retries


System metrics - CPU, memory, threads

Container metrics

Application metrics - latency, number of requests, number of errors

Application logging

Correlation IDs

    - You need them so you can trace the flow of requests and operations through the system.
    - For root cause analysis, if a problem happens in a service, you can find the upstream or downstream calls.
    - For performance analysis, analyze the entire flow
    - A/B testing, be able to analyze metrics for experimental features
    - Pass them in HTTP headers OR put them in message metadata
    - It's good when correlation headers show parent/child relationships so that you can see the hierarchy of calls
    - How to generate correlation IDs?
        - Custom code. Each service looks for the header incoming. If not there, generate it. Then pass it along
        - AI SDK injects 
        

    Example: https://github.com/openzipkin/b3-propagation

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

Latency for real-time monitoring and alerts. How "real-time" is real-time data? A few seconds? A minute?

Data fidelity
- How accurate are the statistics? Averages can hide outliers, especially at scale. 

Dashboard
- Do you get a holistic view of the system? 
- If you are writing telemetry data and logs to more than one location, can the dashboard show all of them and correlate?


## Technical options

Application Insights

    Throttles after 32k/sec. To avoid this, use sampling.
        - Throttling can lead to lost data. (Client SDKs may try to resend)
    Sampling - can be done in the server app (Adaptive sampling) or in AI (ingestion sampling)
        - Ingestion sampling reduces storage but not telemetry traffic
    - There is also a daily cap - this is only meant as a final protection against too much data

Prometheus - time series database 


## Best practices

AI

Log all services to the same AI instance and use cloud_RoleName field to filter by service.

https://docs.microsoft.com/en-us/azure/application-insights/app-insights-monitor-multi-role-app

AI uses correlation IDs - All services should propagate this. 

https://docs.microsoft.com/en-us/azure/application-insights/application-insights-correlation

