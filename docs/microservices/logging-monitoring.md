# Logging and monitoring in a microservices application

In any complex application, at some point something will go wrong. The application may experience failures, or become overwhelmed with requests. Logging and monitoring should provide a holistic view of how the system behaves. 

In a microservices architecture, it can be especially challenging to pinpoint the exact cause of errors or performance bottlenecks. A single user operation might span multiple services. Services may hit network I/O limits inside the cluster. A chain of calls across services may cause backpressure in the system, resulting in high latency or cascading failures. Moreover, you generally don't know which node a particular container will run in. Containers placed on the same node may be competing for limited CPU or memory. 

![](./images/monitoring.png)

You can categorize these into metrics and text-based logs. 

*Metrics* are numerical values that can be analyzed. You can use them to observe the system in real time (or close to real time), or to analyze performance trends over time. 

- System metrics, such as CPU, memory, garbage collection, thread count, and so on. Because services run in containers, you need to collect metrics at the container level, not just at the VM level. Fortunately, Kubernetes provides several options for collecting cluster metrics, including Heapster and [kube-state-metrics][kube-state-metrics].
   
- Application metrics. This includes any metrics that relevant to understanding the behavior of a service. Examples include the number of outgoing HTTP requests, request latency, message queue length, or number of transactions processed per second.

- Dependent service metrics. Services inside the cluster may call external services that are outside the cluster, such as managed PaaS services. You can monitor Azure services by using [Azure Monitor](/azure/monitoring-and-diagnostics/monitoring-overview). Third-party services may or may not provide any metrics. If not, you'll have to rely on your own application metrics to track statistics for latency and error rate.

*Logs* are records of events that occur while the application is running. They include things like application logs (trace statements) or web server logs. Logs are primarily useful for forensics and root cause analysis. 

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
    - Avoid any blocking calls to write logs or telemetry data. 

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


Structured logging. It's good for logs to be structured (JSON) rather than raw text - however you may not control all of the logging, other components in the system (Nginx, IIS, whatever) may be creating unstructured logs. One challengs is how to aggregate them.


Separate monitoring components from services as much as possible

Use a logging abstraction that you can plug in a pipeline by configuration

Handle batching and aggregation in the pipeline, not directly in the app code. Pipeline can also enrich the events with extra information (e.g. correlation ID)


## Correlation IDs and distributed tracing

As mentioned, one challenge in microservices is understanding the flow of events across services. A single operation or transaction may involve calls to multiple services. In order reconstruct the entire sequence of steps, each service should propagate a *correlation ID* that acts as a unique identitifer for that operation. The correlation ID enables [distributed tracing](http://microservices.io/patterns/observability/distributed-tracing.html) across services.

The first service that receives a client request should generates the correlation ID. If the service makes an HTTP call to another service, it puts the correlation ID in a request header. Similarly, if the service sends an asynchronous message, it puts the correlation ID into the message. Downstream services continue to propagate the correlation ID, so that it flows through the entire system. In addition, all code that writes application metrics or log events should include the correlation ID.

If errors or exceptions occur, the correlation ID lets you find the upstream or downstream calls that were part of the same operation. Correlation IDs also makes it possible to calculate metrics like end-to-end latency for a complete transaction, number of successful transactions per second, and percentage of failed transactions.

Some considerations when implementing correlation IDs:

- There is currently no standard HTTP header for correlation IDs. Your team should standardize on a custom header value. The choice may be decided by your logging/monitoring framework or the service mesh.

- For asynchronous messages, if your messaging infrastructure supports adding metadata to messages, you should include the correlation ID as metadata. Otherwise, include it as part of the message schema.

- Rather than a single opaque identitier, you might send a *correlation context* that includes richer information, such the caller-callee relationships. 

- The Azure Application Insights SDK will automatically inject correlation IDs into HTTP headers, and includes the correlation ID in Application Insights logs. If you decide to use the correlation features built into Application Insights, some services may still need to explicity propagate the correlation ID. For more information, see [Telemetry correlation in Application Insights](/azure/application-insights/application-insights-correlation).
   
- If you are using Istio or linkerd as a service mesh, these technologies automatically generate correlation headers when HTTP calls are routed through the service mesh proxies. Services should forward the relevant headers. 

    - Istio: [Distributed Request Tracing](https://istio-releases.github.io/v0.1/docs/tasks/zipkin-tracing.html)
    
    - linkerd: [Context Headers](https://linkerd.io/config/1.3.0/linkerd/index.html#http-headers)
    
- Consider how you will aggregate logs. You may want to standardize on a schema for including correlation IDs in logs across all services.


## Technical options

**Kubernetes** collects cluster metrics. You can use a service like Heapster or [kube-state-metrics][kube-state-metrics] to aggregate these metrics and send them to a sink.

**Application Insights** is a managed service in Azure that ingests and stores telemetry data, and provides tools for analyzing and searching the data. To use Application Insights, you install an instrumentation package in your application. This packages monitors the app and sends telemetry data to the Application Insights service. It can also pull telemetry data from the host environment. 



    Throttles after 32k/sec. To avoid this, use sampling.
        - Throttling can lead to lost data. (Client SDKs may try to resend)
    Sampling - can be done in the server app (Adaptive sampling) or in AI (ingestion sampling)
        - Ingestion sampling reduces storage but not telemetry traffic
    - There is also a daily cap - this is only meant as a final protection against too much data

    Application Insights will automatically go into adaptive sampling mode when ingest throughput is exceeded (this will cap at 32k ops/sec for the largest plan).  This will result in accurate metrics, but only a portion of the raw data being captured.  
    
    Built in correlation features    


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


<!-- links -->

[kube-state-metrics]: https://github.com/kubernetes/kube-state-metrics
