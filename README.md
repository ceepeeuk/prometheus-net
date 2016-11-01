# Prometheus .NET Client

This is an experimental dot net core version (unofficial).

Its a fork of [MihaMarkic's](https://github.com/MihaMarkic/prometheus-net) fork of prometheus-net, updated to work with dotnet core and xunit.

I have removed completely the MetricServer and suggest use of a /metrics controller, containing code such as:

```csharp
using System.Linq;
using Microsoft.AspNetCore.Mvc;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Prometheus;
using Prometheus.Advanced;

namespace ReferenceFileService.Controllers
{
	[Route("api/[controller]")]
	public class MetricsController: Controller
    {
		[HttpGet]
		public IActionResult Get()
		{
			var registry = DefaultCollectorRegistry.Instance;
			var acceptHeaders = Request.Headers["Accept"];
			var contentType = ScrapeHandler.GetContentType(acceptHeaders);
			Response.ContentType = contentType;
			var s = ScrapeHandler.ProcessScrapeRequest(registry.CollectAll(), contentType);
			return new OkObjectResult(s);
		}
    }
}

```
Tested on dotnet core netstandard1.6.

See prometheus [here](http://prometheus.io/)

# Installation

## Instrumenting

Four types of metric are offered: Counter, Gauge, Summary and Histogram.
See the documentation on [metric types](http://prometheus.io/docs/concepts/metric_types/)
and [instrumentation best practices](http://prometheus.io/docs/practices/instrumentation/#counter-vs.-gauge-vs.-summary)
on how to use them.

### Counter

Counters go up, and reset when the process restarts.


```csharp
var counter = Metrics.CreateCounter("myCounter", "some help about this");
counter.Inc(5.5);
```

### Gauge

Gauges can go up and down.


```csharp
var gauge = Metrics.CreateGauge("gauge", "help text");
gauge.Inc(3.4);
gauge.Dec(2.1);
gauge.Set(5.3);
```

### Summary

Summaries track the size and number of events.

```csharp
var summary = Metrics.CreateSummary("mySummary", "help text");
summary.Observe(5.3);
```

### Histogram

Histograms track the size and number of events in buckets.
This allows for aggregatable calculation of quantiles.

```csharp
var hist = Metrics.CreateHistogram("my_histogram", "help text", buckets: new[] { 0, 0.2, 0.4, 0.6, 0.8, 0.9 });
hist.Observe(0.4);
```

The default buckets are intended to cover a typical web/rpc request from milliseconds to seconds.
They can be overridden passing in the `buckets` argument.

### Labels

All metrics can have labels, allowing grouping of related time series.

See the best practices on [naming](http://prometheus.io/docs/practices/naming/)
and [labels](http://prometheus.io/docs/practices/instrumentation/#use-labels).

Taking a counter as an example:

```csharp
var counter = Metrics.CreateCounter("myCounter", "help text", labelNames: new []{ "method", "endpoint"});
counter.Labels("GET", "/").Inc();
counter.Labels("POST", "/cancel").Inc();
```

## HTTP handler

Metrics are usually exposed over HTTP, to be read by the Prometheus server.

```csharp
var metricServer = new MetricServer(port: 1234);
metricServer.Start();
```

## Unit testing
For simple usage the API uses static classes, which - in unit tests - can cause errors like this: "A collector with name '<NAME>' has already been registered!"

To address this you can add this line to your test setup:

```csharp
DefaultCollectorRegistry.Instance.Clear();
```
