---
title: Getting started
sort_rank: 3
---

# Getting started

This guide is a "Hello World"-style tutorial which shows how to install,
configure, and use Prometheus in a simple example setup. You will download and run
Prometheus locally, configure it to scrape itself and an example application,
and then work with queries, rules, and graphs to make use of the collected time
series data.

## Downloading and running Prometheus

[Download the latest release](https://github.com/prometheus/prometheus/releases)
of Prometheus for your platform, then extract and run it:

```
tar xvfz prometheus-*.tar.gz
./prometheus
```

It should fail to start, complaining about the absence of a configuration file.

## Configuring Prometheus to monitor itself

Prometheus collects metrics from monitored targets by scraping metrics HTTP
endpoints on these targets. Since Prometheus also exposes data in the same
manner about itself, it can also scrape and monitor its own health.

While a Prometheus server that collects only data about itself is not very
useful in practice, it is a good starting example. Save the following basic
Prometheus configuration as a file named `prometheus.yml`:

```
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # By default, scrape targets every 15 seconds.
  # scrape_timeout is set to the global default (10s).

  # Attach these extra labels to all timeseries collected by this Prometheus instance.
  labels:
    monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    scrape_timeout: 10s

    target_groups:
      - targets: ['localhost:9090']
```

For a complete specification of configuration options, see the
[configuration documentation](/docs/operating/configuration).


## Starting Prometheus

To start Prometheus with your newly created configuration file, change to your
Prometheus build directory and run:

```language-bash
# Start Prometheus.
# By default, Prometheus stores its database in ./data (flag -storage.local.path).
./prometheus -config.file=prometheus.yml
```

Prometheus should start up and it should show a status page about itself at
http://localhost:9090. Give it a couple of seconds to collect data about itself
from its own HTTP metrics endpoint.

You can also verify that Prometheus is serving metrics about itself by
navigating to its metrics endpoint: http://localhost:9090/metrics

By default, Prometheus will only execute at most one OS thread at a
time. In production scenarios on multi-CPU machines, you will most
likely achieve better performance by setting the `GOMAXPROCS`
environment variable to a value similar to the number of available CPU
cores:

```language-bash
GOMAXPROCS=8 ./prometheus -config.file=prometheus.yml
```

Blindly setting `GOMAXPROCS` to a high value can be
counterproductive. See the relevant [Go
FAQs](http://golang.org/doc/faq#Why_no_multi_CPU).

## Using the expression browser

Let us try looking at some data that Prometheus has collected about itself. To
use Prometheus's built-in expression browser, navigate to
http://localhost:9090/graph and choose the "Tabular" view within the "Graph"
tab.

As you can gather from http://localhost:9090/metrics, one metric that
Prometheus exports about itself is called
`prometheus_target_interval_length_seconds` (the actual amount of time between
target scrapes). Go ahead and enter this into the expression console:

```
prometheus_target_interval_length_seconds
```

This should return a lot of different time series (along with the latest value
recorded for each), all with the metric name
`prometheus_target_interval_length_seconds`, but with different labels. These
labels designate different latency percentiles and target group intervals.

If we were only interested in the 99th percentile latencies, we could use this
query to retrieve that information:

```
prometheus_target_interval_length_seconds{quantile="0.99"}
```

To count the number of returned time series, you could write:

```
count(prometheus_target_interval_length_seconds)
```

For more about the expression language, see the
[expression language documentation](/docs/querying/basics/).

## Using the graphing interface

To graph expressions, navigate to http://localhost:9090/graph and use the "Graph"
tab.

For example, enter the following expression to graph the per-second rate of all
storage chunk operations happening in the self-scraped Prometheus:

```
rate(prometheus_local_storage_chunk_ops_total[1m])
```

Experiment with the graph range parameters and other settings.

## Starting up some sample targets

Let us make this more interesting and start some example targets for Prometheus
to scrape.

The Go client library includes an example which exports fictional RPC latencies
for three services with different latency distributions.

Ensure you have the [Go compiler installed](https://golang.org/doc/install). 
Download the Go client library for Prometheus and run three of these example
processes:

```bash
# Fetch the client library code and compile example.
git clone https://github.com/prometheus/client_golang.git
cd client_golang/examples/random
go build

# Start 3 example targets in separate terminals:
./random -listen-address=:8080
./random -listen-address=:8081
./random -listen-address=:8082
```

You should now have example targets listening on http://localhost:8080/metrics,
http://localhost:8081/metrics, and http://localhost:8082/metrics.

## Configuring Prometheus to monitor the sample targets

Now we will configure Prometheus to scrape these new targets. Let's group all
three endpoints into one job called `example-random`. However, imagine that the
first two endpoints are production targets, while the third one represents a
canary instance. To model this in Prometheus, we can add several groups of
endpoints to a single job, adding extra labels to each group of targets. In
this example, we will add the `group="production"` label to the first group of
targets, while adding `group="canary"` to the second.

To achieve this, add the following job definition to the `scrape_configs`
section in your `prometheus.yml` and restart your Prometheus instance:

```
scrape_configs:
  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    scrape_timeout: 10s

    target_groups:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

Go to the expression browser and verify that Prometheus now has information
about time series that these example endpoints expose, such as the
`rpc_durations_microseconds` metric.

## Configure rules for aggregating scraped data into new time series

Though not a problem in our example, queries that aggregate over thousands of
time series can get slow when computed ad-hoc. To make this more efficient,
Prometheus allows you to prerecord expressions into completely new persisted
time series via configured recording rules. Let's say we are interested in
recording the per-second rate of example RPCs
(`rpc_durations_microseconds_count`) averaged over all instances (but
preserving the `job` and `service` dimensions) as measured over a window of 5
minutes. We could write this as:

```
avg(rate(rpc_durations_microseconds_count[5m])) by (job, service)
```

Try graphing this expression.

To record the time series resulting from this expression into a new metric
called `job_service:rpc_durations_microseconds_count:avg_rate5m`, create a file
with the following recording rule and save it as `prometheus.rules`:

```
job_service:rpc_durations_microseconds_count:avg_rate5m = avg(rate(rpc_durations_microseconds_count[5m])) by (job, service)
```

To make Prometheus pick up this new rule, add a `rule_files` statement to the
global configuration section in your `prometheus.yml`. The config should now
look like this:

```
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # By default, scrape targets every 15 seconds.
  # scrape_timeout is set to the global default (10s).

  # Attach these extra labels to all timeseries collected by this Prometheus instance.
  labels:
    monitor: 'codelab-monitor'

rule_files:
  - 'prometheus.rules'

scrape_configs:
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    scrape_timeout: 10s

    target_groups:
      - targets: ['localhost:9090']

  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    scrape_timeout: 10s

    target_groups:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

Restart Prometheus with the new configuration and verify that a new time series
with the metric name `job_service:rpc_durations_microseconds_count:avg_rate5m`
is now available by querying it through the expression browser or graphing it.
