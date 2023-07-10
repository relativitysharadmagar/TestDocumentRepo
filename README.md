##  APM Logging: Document Review Service
Creates Application Performance Monitoring (APM) related objects.
To monitor Conversion Cache Manager operations in Relativity One. At this time APM Metrics are being introduced to monitor job creation, discovery, and deleting files.

### Registered Services
ConversionApmClient

### Configurator
Need to install below .NETStandard,Version=2.0 supprting APM nuget.
```
Relativity.Telemetry.APM   Version="2019"
```
#### Dependency Registration
Code Snippet: ServiceInstaller.cs
```
services.AddTransient<IConversionApmClient, Review.Shared.ConversionApmClient>();
```
#### Startup Configuration
Code Snippet: Startup.cs
```
services.AddSingleton<IAPM>(x => new Instrumentation(_configuration).APM);
services.AddOpenTelemetry(_configuration.GetSection(RuntimeConfiguration.Position).Get<RuntimeConfiguration>());
```
#### Instrumentation Class
Instrumentation class contains the k8s and EventHub configuration.
##### K8s Configuration
Code Snippet: Instrumentation.cs
```
var customData = new ConcurrentDictionary<string, object>();
		 customData.TryAdd("k8s.cluster.name", _configuration.GetValue<string>("K8S_CLUSTER_NAME"));
		 customData.TryAdd("k8s.pod.name", Environment.MachineName);
		 customData.TryAdd("service.version", ServiceConfiguration.ASSEMBLY_VERSION);
```
##### EventHubConfig
Code Snippet: Instrumentation.cs
```
ar config = new EventHubConfig(
		eventHubServiceNamespace: () => _configuration.GetValue<string>("R1_REGION_APM_NAMESPACE"),
		eventHubName: () => _configuration.GetValue<string>("R1_REGION_APM_NAME"),
		eventHubSharedAccessKeyName: () => _configuration.GetValue<string>("R1_REGION_APM_KEYNAME"),
		eventHubSharedAccessKey: () => _configuration.GetValue<string>("R1_REGION_APM_KEY"));
```

#### ObservabilityMiddleware
ObservabilityMiddleware is a middleware which provides the OpenTelemetry resources, metrics and tracing.

Code Snippet: ObservabilityMiddleware.cs
```
services.AddHostedService<OpenTelemetryEventListener>();
	 OpenTelemetryBuilder otelBuilder = services.AddOpenTelemetry();
	 otelBuilder.AddOpenTelemetryResource(configuration);
	 otelBuilder.AddOpenTelemetryMetrics(configuration);
	 otelBuilder.AddOpenTelemetryTracing(configuration);
```
#### Constructor Injection
Code Snippet: DocumentManager.cs
```
public DocumentManager(IConversionApmClient apmClient)
{
	_apmClient = apmClient;
}
```
### Calling APM Methods
Code Snippet: DocumentManager.cs
```
_apmClient.ReportTimeToApm(PerformanceKeys.DVS_DOCUMENT_MANAGER_HANDLE_TIME, stopwatch.ElapsedMilliseconds, metrics);
_apmClient.LogDocToDocCacheHitMetric(batchRequest, false);
_apmClient.ReportErrorToApm(ErrorType.LongAuditInsert, Guid.Empty, Guid.NewGuid(), ex, batchRequest.WorkspaceId);
```
### How APM Works
The APM Framework consists of two basic operations: collecting metrics of interest and sending them on to their destination.

### The Measures
There are 5 basic measure types available in the new APM framework that cover most use cases for capturing telemetry and APM information at the application level.  The APM library provides five types of measures that can be recorded (see Metric Types for a more detailed overview).

#### Meters
 Records the rate at which an event occurs

#### Health Check
Measure the health of a system

#### Timers
Keeps a histogram of the duration of a type of event

#### Counters
Counters are 64 bit integers that can be incremented or decremented to count the occurrences of an event or value

#### Gauges
Instantaneous values (there is also a variation that allows for capturing gauge values at timed intervals)

Metrics can be created and registered using the methods available on the APM class, which is available in the Telemetry namespace. Creating an APM Client allows for developers to retrieve any of the measures that make sense for the type of information to be captured. The individual measures then forward their collected values to one or more sinks (destinations) for further processing and visualization (graphing & alerting). At the current time, the APM framework supports multiple sinks for receiving generated metrics. The default sink for both RelativityOne and On Premise solutions is the Relativity ServiceBus. This allows metrics to be sent to a remote processing solution where graphing ,alerting, and further processing can occur without effecting the running instance of Relativity.

### New Relic Query Verification
#### Guage
```
SELECT * FROM Gauge WHERE Name = 'DocumentReview' and CustomData_id = 'd851f0b3-a877-b4db-2474-9717a63d69g6' since 15 minutes ago
```

#### Timer
```
SELECT * from Timer where Name = 'DVS.DocumentManagerHandleTime' and CustomData_viewerContentKeyRequestPriority = 'OnTheFly' and CustomData_MaaSEnvironmentType = 'regression' since 5 minutes ago
```
