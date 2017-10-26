# Logging and monitoring

Monitoring to give you a holistic view of how the system is performing. Identify bottlenecks or services that are having problems. Services may be resource constrained, experiencing failures, or overwhelmed with requests.

Some issues that are common to MSA
- Network I/O limitations within the cluster
- Resource constraints due to running multiple containers on a single node, or containers with resource caps. 
- Backpressure. A service has a queue of requests it is processing, that causes upstream services to back up.
- Latency caused cascading retries


Why is monitoring more challenging in MSA?
- Single request goes through multiple services
- Bottlenecks harder to pinpoint
- You generally don't know which node a particular container will run in
- You need to collect metrics at the container level, the node level, the cluster level


Kubernetes controllers will monitor the system for pod health and number of replicas.

Kubelet collects stats







Types of logs to collect:

    System metrics - CPU, memory, threads
    
    Container metrics
    
    Application metrics - latency, number of requests, number of errors
    
    Application logging
    
    Dependent service metrics
    
    Structured logging
    
    Client metrics
    
You can categorize these into metrics and text-based logs. Metrics are numerical values that can be analyzed, find averages and trends. Text based logs are good for root cause analysis, debugging, forensics. 

It's good for logs to be structured (JSON) rather than raw text - however you may not control all of the logging, other components in the system (Nginx, IIS, whatever) may be creating unstructured logs. One challengs is how to aggregate them.


Correlation IDs

    - You need them so you can trace the flow of requests and operations through the system.
    - For root cause analysis, if a problem happens in a service, you can find the upstream or downstream calls.
    - For performance analysis, analyze the entire flow
    - A/B testing, be able to analyze metrics for experimental features
    - Pass them in HTTP headers OR put them in message metadata
        - There are currently no standard headers for correlation IDs
    - It's good when correlation headers show parent/child relationships so that you can see the hierarchy of calls
    - How to generate correlation IDs?
        - Custom code. Each service looks for the header incoming. If not there, generate it. Then pass it along. Include in logging statements
        - AI SDK injects into headers and into logging
        - Service Mesh adds to headers and in service mesh logging
        - Process: (a) generate the ID, (b) pass along IDs in HTTP headers and include IDs in async messages, (c) Include the correlation ID in logging statements. Depending on frameworks/libraries, some of this may be done for you
        - Consider how you will aggregate logs - you may want to standardize on schema for correlation ID in logs. Use structured logging (JSON)

    Example: https://github.com/openzipkin/b3-propagation



## Considerations

Configuration and management
    - Managed service vs deployed in the cluster
    - You can do monitoring inside the cluster and also push to OMS 

Ingestion rate

    - What is the "raw" rate at which the system can intake telemetry events?
    - If the rate is exceeded, what happens? (Throttling, downsampling)
    - Approaches to reduce the load on the telemetry system:
        - Aggregation of metrics (statistical data)
        - Sampling - process only a percentage of the events
        - Batching - to reduce the number of calls to the telemetry service

    - What is the granularity of the data. 

What can it log 
    - Prometheus only supports floats, not strings. 
    - 

Storage costs

    - Date should eventually be moved to long-term storage so it is available for retrospective analysis

Resiliency - where is the data stored, can it be lost if a node goes down?

Latency for real-time monitoring and alerts. How "real-time" is real-time data? A few seconds? A minute?

Data fidelity
- How accurate are the statistics? Averages can hide outliers, especially at scale. 

Dashboard
- Do you get a holistic view of the system? 
- If you are writing telemetry data and logs to more than one location, can the dashboard show all of them and correlate?


Separate monitoring components from services as much as possible

Use a logging abstraction that you can plug in a pipeline by configuration

Handle batching and aggregation in the pipeline, not directly in the app code. Pipeline can also enrich the events with extra information (e.g. correlation ID)


## Technical options

Application Insights

    Throttles after 32k/sec. To avoid this, use sampling.
        - Throttling can lead to lost data. (Client SDKs may try to resend)
    Sampling - can be done in the server app (Adaptive sampling) or in AI (ingestion sampling)
        - Ingestion sampling reduces storage but not telemetry traffic
    - There is also a daily cap - this is only meant as a final protection against too much data

    Application Insights will automatically go into adaptive sampling mode when ingest throughput is exceeded (this will cap at 32k ops/sec for the largest plan).  This will result in accurate metrics, but only a portion of the raw data being captured.  
    

Prometheus - time series database 
    - Integrated with kubernetes
    - linkerd / istio integration
    - Does not support string data, only floats

Influxdb

    influxdb allow automatic downsampling to retain old data at a lower fidelity (masimms)
    Supports string data + metrics
    Horizontal scaling is only supported in the commercial version

Grafana - dashboard

Zipkin? - distributed tracing



## Our architecture

System metrics captured by kubernetes and sent to prometheus with grafana
    - Prometheus log apis for app metrics and exporter for system metrics 

Application logs go to standard out, kubelet (?) writes to local file system, fluentd ships the logs to elasticsearch running in the cluster



## Best practices

Use labels to distinguish types of pods (front end, application, gateway, infrastructure services)





AI

Log all services to the same AI instance and use cloud_RoleName field to filter by service.

https://docs.microsoft.com/en-us/azure/application-insights/app-insights-monitor-multi-role-app

AI uses correlation IDs - All services should propagate this. 

https://docs.microsoft.com/en-us/azure/application-insights/application-insights-correlation

Using AI at scale: https://blogs.msdn.microsoft.com/azurecat/2017/05/11/azure-monitoring-and-analytics-at-scale/
