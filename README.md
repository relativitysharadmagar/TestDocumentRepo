##  Using Relativity APM Client: Document Review Service
Creates Application Performance Monitoring (APM) related objects.
To continue to report existing APM to match the APM that would have come from Kepler.
### Registered Services
Usages of IAPMClient

### Configurator
Need to install below .NETStandard,Version=2.0 supprting APM nuget.
```
Relativity.Telemetry.APM   Version="2019"
```
#### Dependency Registration
Code Snippet: ServiceInstaller.cs  
Source Code: https://github.com/relativityone/ReviewService/blob/main/Source/reviewservice/IoC/ServicesInstaller.cs
```
services.AddTransient<IConversionApmClient, Review.Shared.ConversionApmClient>();
```
#### Startup Configuration
Code Snippet: Startup.cs  
Source Code: https://github.com/relativityone/ReviewService/blob/main/Source/reviewservice/Startup.cs
```
services.AddSingleton<IAPM>(x => new Instrumentation(_configuration).APM);
services.AddOpenTelemetry(_configuration.GetSection(RuntimeConfiguration.Position).Get<RuntimeConfiguration>());
```
#### Instrumentation Class
Instrumentation class contains the k8s and EventHub configuration.
##### K8s Configuration
Read the config value K8S_CLUSTER_NAME from the appsettings.Development.json file.  
Code Snippet: Instrumentation.cs  
Source Code: https://github.com/relativityone/ReviewService/blob/main/Source/reviewservice/Middleware/Observability/Instrumentation.cs
```
var customData = new ConcurrentDictionary<string, object>();
		 customData.TryAdd("k8s.cluster.name", _configuration.GetValue<string>("K8S_CLUSTER_NAME"));
		 customData.TryAdd("k8s.pod.name", Environment.MachineName);
		 customData.TryAdd("service.version", ServiceConfiguration.ASSEMBLY_VERSION);
```
Code Snippet: appsettings.Development.json  
Source Code: https://github.com/relativityone/ReviewService/blob/main/Source/reviewservice/appsettings.Development.json
```
{
    "Runtime": {
		"OTEL_EXPORTER_OTLP_ENDPOINT": "xxxxxxx",
		"CID_AUTHORITY_URI": "https://xxxxxxxxxxxxx/",
		"K8S_CLUSTER_NAME": "xxxxxx",
		"SAMPLING_RATE": 0.0,
		"K8S_REGION_NAME": "xxx",
		"ENABLE_XHOST_OVERRIDE_HEADER":  true
	       }
}
```
##### EventHubConfig Configuration
Read the below config value from the secrets.json file.
Code Snippet: Instrumentation.cs  
Source Code: https://github.com/relativityone/ReviewService/blob/main/Source/reviewservice/Middleware/Observability/Instrumentation.cs
```
ar config = new EventHubConfig(
		eventHubServiceNamespace: () => _configuration.GetValue<string>("R1_REGION_APM_NAMESPACE"),
		eventHubName: () => _configuration.GetValue<string>("R1_REGION_APM_NAME"),
		eventHubSharedAccessKeyName: () => _configuration.GetValue<string>("R1_REGION_APM_KEYNAME"),
		eventHubSharedAccessKey: () => _configuration.GetValue<string>("R1_REGION_APM_KEY"));
```
Code Snippet: secrets.json  
Source Code: Find your local directory secrets.json file
```
{
  "Runtime:R1_REGION_APM_NAME": "xxxx-xxxxxx-xxx-xxx",
  "Runtime:R1_REGION_APM_KEYNAME": "xxxxxxxx",
  "Runtime:R1_REGION_APM_KEY": "xxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "Runtime:R1_REGION_APM_NAMESPACE": "xxx-xx-xxx-xxx"
}
```

#### ObservabilityMiddleware Configuration
ObservabilityMiddleware is a middleware which provides the OpenTelemetry resources, metrics and tracing.  
Code Snippet: ObservabilityMiddleware.cs  
Source Code: https://github.com/relativityone/ReviewService/blob/main/Source/reviewservice/Middleware/Observability/ObservabilityMiddleware.cs
```
services.AddHostedService<OpenTelemetryEventListener>();
	 OpenTelemetryBuilder otelBuilder = services.AddOpenTelemetry();
	 otelBuilder.AddOpenTelemetryResource(configuration);
	 otelBuilder.AddOpenTelemetryMetrics(configuration);
	 otelBuilder.AddOpenTelemetryTracing(configuration);
```
#### Constructor Injection
Code Snippet: DocumentManager.cs  
Source Code: https://github.com/relativityone/ReviewService/blob/main/Source/Relativity.DocumentViewer.Core/Managers/DocumentManager.cs  
```
public DocumentManager(IConversionApmClient apmClient)
{
	_apmClient = apmClient;
}
```
### Calling APM Methods
Code Snippet: DocumentManager.cs  
Source Code: https://github.com/relativityone/ReviewService/blob/main/Source/Relativity.DocumentViewer.Core/Managers/DocumentManager.cs
```
_apmClient.ReportTimeToApm(PerformanceKeys.DVS_DOCUMENT_MANAGER_HANDLE_TIME, stopwatch.ElapsedMilliseconds, metrics);
_apmClient.LogDocToDocCacheHitMetric(batchRequest, false);
_apmClient.ReportErrorToApm(ErrorType.LongAuditInsert, Guid.Empty, Guid.NewGuid(), ex, batchRequest.WorkspaceId);
```
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
Once this code has been migrated, we should be able to validate that the same APM that was reported in Kepler is now being reported in the HA service.
### New Relic Query Verification
#### Guage
Pass the guage name and custom data id for the mentioned guage service and pod in the NewRelic Query.
```
SELECT * FROM Gauge WHERE Name = 'DocumentReview' and CustomData_id = 'xxxx-xxxx-xxxx-xxxx-xxxxxxxx' since 15 minutes ago
```
Code Snippet: NewRelic Query to retrieve the custom data id.
```
SELECT COUNT(1) as Count,Timestamp,CorrelationId,CustomData_id,CustomData_MaaSMessageProcessedTime,CustomData_state,SourceName,Timestamp  FROM Counter WHERE Name='em_job_state_change_total' FACET SourceName
```
#### Timer
Pass the timer name for your manager and conetent key request priority to the NewRelic Query.
```
SELECT * from Timer where Name = 'YOUR_MANAGER_HANDLE_TIME_NAME' and CustomData_viewerContentKeyRequestPriority = 'PRIORITY_NAME' and CustomData_MaaSEnvironmentType = 'ENVIRONMENT_NAME' since 5 minutes ago
```
