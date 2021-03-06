# Prometheus

- [Prometheus](#prometheus)
  - [Part 01: The Basics](#part-01-the-basics)
    - [Prometheus Architecture](#prometheus-architecture)
    - [Prometheus Limitation](#prometheus-limitation)
  - [Part 02: Installation and Configuration](#part-02-installation-and-configuration)
    - [Installation Options:](#installation-options)
    - [Configuration](#configuration)
  - [Part 03: Prometheus Data (Data Model for Storing Data + Query Language to Interact with Prometheus Data)](#part-03-prometheus-data-data-model-for-storing-data--query-language-to-interact-with-prometheus-data)
    - [Metric Names](#metric-names)
    - [Metric Labels](#metric-labels)
    - [Metric Types](#metric-types)
    - [Querying](#querying)
      - [Selectors](#selectors)
  - [Part 04: Visualization](#part-04-visualization)
  - [Part 05: Collecting Metrics](#part-05-collecting-metrics)
  - [Part 06: Alerting](#part-06-alerting)
  - [Part 07: Advanced Concepts (HA and Federation)](#part-07-advanced-concepts-ha-and-federation)
  - [Part 08: Security](#part-08-security)
  - [Part 09: Client Libraries](#part-09-client-libraries)
  - [Part 10: References](#part-10-references)

---
## Part 01: The Basics
[Prometheus](https://prometheus.io) is an open-source **monitoring** and **alerting** tool. It collects data about applications and systems and allows you to **visualize** the data and issue alerts based on the data.

**Monitoring** is an important component of DevOps automation. To manage a robust and complex infrastructure, you need to be able to quickly and easily understand what is happening in your systems.

High Level Use Cases:
- **Metric Collection**: Collect important metrics about your systems and applications in one place.
- **Visualization**: Build dashboards that provide an overview of the health of your systems.
- **Alerting**: Send you an email or mobile notification when something is broken.

### Prometheus Architecture

Prometheus Architecture Diagram:

![Architecture](Images/architecture.png)

The components of a Prometheus system are:
- **Prometheus Server**: A central server that gathers metrics and makes them available.
- **Exporters**: Agents that expose data about systems and applications for collecting by the Prometheus server.
- **Client Libraries**: Easily turn your custom application into an exporter that exposes metrics in a format Prometheus can consume.
- **Prometheus Pushgateway**: Allows pushing metrics to Prometheus for certain specific use cases. Pushgateway acts as middle-man.
- **Alert Manager**: Sends alerts triggered by metric data.
- **Visualization Tools**: Provide useful ways to view metric data. These are not necessarily part of Prometheus.

Prometheus collects metrics using a **pull model**. This means the Prometheus server pulls metric data from exporters - agents do not push data to the Prometheus server.

### Prometheus Limitation

While Prometheus is a great tool for a variety of use cases, it is important to understand when it is not the best tool:
- **100% Accuracy** (e.g., per-request billing): Prometheus is designed to operate even under failure conditions. This means it will continue to provide data even if new data is not available due to failure and outages. If you need 100% up-to-the-minute accuracy, such as in the case of per request billing, Prometheus may not be the best tool to use.
- **Non Time-Series Data** (e.g., log aggregation): Prometheus is built to monitor time-series metrics, especially data that is numeric. It is not the best choice for collecting more generic types of data such as system logs.

---
## Part 02: Installation and Configuration

### Installation Options:
1. Use pre-compiled binaries from [here](https://prometheus.io/download/).
    ```bash
    #!/bin/bash

    sudo useradd -M -r -s /bin/false prometheus
    sudo mkdir /etc/prometheus /var/lib/prometheus
    # Now download the latest version
    tar xzf PROMETHEUS.tar.gz
    sudo cp PROMETHEUS/{prometheus,promtool} /usr/local/bin/
    sudo chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}
    sudo cp -r PROMETHEUS/{consoles,console_libraries} /etc/prometheus/
    sudo cp PROMETHEUS/prometheus.yml /etc/prometheus/prometheus.yml
    sudo chown -R prometheus:prometheus /etc/prometheus
    sudo chown prometheus:prometheus /var/lib/prometheus
    # Test prometheus: prometheus --config.file=/etc/prometheus/prometheus.yml
    
    # We need to turn prometheus to systemd service
    sudo cat >> /etc/systemd/system/prometheus.service << EOF
    [Unit]
    Description=Prometheus Time Series Collection and Processing Server
    Wants=network-online.target
    After=network-online.target

    [Service]
    User=prometheus
    Group=prometheus
    Type=simple
    ExecStart=/usr/local/bin/prometheus \
        --config.file /etc/prometheus/prometheus.yml \
        --storage.tsdb.path /var/lib/prometheus/ \
        --web.console.templates=/etc/prometheus/consoles \
        --web.console.libraries=/etc/prometheus/console_libraries
    ExecReload=/bin/kill -HUP $MAINPID

    [Install]
    WantedBy=multi-user.target
    EOF
    
    sudo systemctl daemon-reload
    sudo systemctl start prometheus
    sudo systemctl enable prometheus
    # Test prometheus service: sudo systemctl enable prometheus
    # The default port of prometheus is 9090
    ```
2. Build from [source](https://github.com/prometheus).
3. Run with Docker using pre-built [images](https://hub.docker.com/r/prom/prometheus/).

### Configuration

- Prometheus configuration file format is YAML.
- Reloading the Configuration:
  1. Restarting Prometheus: `systemctl restart prometheus`
  2. Reloading Prometheus: `systemctl reload prometheus`
  3. Send SIGHUP to Prometheus Process: `killall -HUP prometheus`
- Read configuration file from URL: `curl localhost:9090/api/v1/status/config`

Exporters:
- A Prometheus exporter is any application that exposes metric data in a format that can be controlled (or "scraped") by the Prometheus server.
- Configure Node Exporter:
  ```bash
  # Create a user on the server you want to monitor
  sudo useradd -M -r -s /bin/false node_exporter

  # Download Node Exporter Binary from Prometheus website and extract it
  wget "LINK"
  tar xvzf node_exporter.tar.gz

  # Copy the binary file to suitable location for the system
  sudo cp node_exporter /usr/local/bin/

  # Change ownership of the binary to node_exporter user
  sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

  # Create a system service for the node_exporter
  sudo cat >> /etc/systemd/system/node_exporter.service << EOF
  [Unit]
  Description= Prometheus Node Exporter
  Wants=network-online.target
  After=network-online.target

  [Service]
  User=node_exporter
  Group=node_exporter
  Type=simple
  ExecStart=/usr/local/bin/node_exporter

  [Install]
  WantedBy=multi-user.target
  EOF

  sudo systemctl daemon-reload
  sudo systemctl start node_exporter
  sudo systemctl enable node_exporter

  # Check connection of the Node Exporter
  curl localhost:9100/metrics
  ```
- Create a service for init based servers (Centos 6)
  ```bash
  # Command Usage: sudo service node_exporter status|stop|start
  sudo cat >> /etc/init.d/node_exporter << EOF
  #!/bin/bash
  OPTIONS=`cat /etc/sysconfig/node_exporter`
  RETVAL=0
  PROG="node_exporter"
  EXEC="/usr/local/bin/node_exporter"
  LOCKFILE="/var/lock/subsys/$PROG"
  LOGFILE=/var/log/node_exporter.log
  ErrLOGFILE=/var/log/node_exporter_error.log

  # Source function library.
  if [ -f /etc/rc.d/init.d/functions ]; then
    . /etc/rc.d/init.d/functions
  else
    echo "/etc/rc.d/init.d/functions is not exists"
    exit 0
  fi

  start() {
    if [ -f $LOCKFILE ]
    then
      echo "$PROG is already running!"
    else
      echo -n "Starting $PROG: "
      nohup $EXEC $OPTIONS > $LOGFILE 2> $ErrLOGFILE &
      RETVAL=$?
      [ $RETVAL -eq 0 ] && touch $LOCKFILE && success || failure
      echo
      return $RETVAL
    fi
  }

  stop() {
    echo -n "Stopping $PROG: "
    killproc $EXEC
    RETVAL=$?
    [ $RETVAL -eq 0 ] && rm -r $LOCKFILE && success || failure
    echo
  }

  restart ()
  {
    stop
    sleep 1
    start
  }

  case "$1" in
    start)
      start
      ;;
    stop)
      stop
      ;;
    status)
      status $PROG
      ;;
    restart)
      restart
      ;;
    *)
      echo "Usage: $0 {start|stop|restart|status}"
      exit 1
  esac
  exit $RETVAL
  ```

Scrape Config:
- The *scrape_config* section of the Prometheus config file provides a list of targets the Prometheus server will scrape, such as a Node Exporter running on a Linux machine.
- Prometheus server will scrape these targets periodically to collect metric data.
- We need to add the Node Exporter specification to Prometheus configuration file:
  ```yaml
  # Start of /etc/prometheus/prometheus.yml config file
  .
  .
  .
  scrape_configs:
    # Prometheus server
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
    # Node Exporter Servers
    - job_name: 'NAME_OF_SERVER'
      static_configs:
      - targets: ['PRIVATE_IP:9100']
  .
  .
  .
  # End of /etc/prometheus/prometheus.yml config file
  ```
- Don't forget to restart/reload the prometheus service.

---
## Part 03: Prometheus Data (Data Model for Storing Data + Query Language to Interact with Prometheus Data)

- Prometheus is built around storing time-series data.
- Time-series data consists of a series of values associated with different points in time.
- Every metric in Prometheus tracks a particular value over time.

### Metric Names
Every metric in Prometheus has a metric name. THe metric name refers to the general feature of a system or application that is being measured.

An example of a metric name: node_cpu_seconds_total

Note that the metric name merely refers to the feature being measured. Metric names do not point to a specific data value but potentially a collection of many values.

### Metric Labels
Prometheus uses labels to provide a dimensional data model. This means we can use labels to specify additional things, such as which node's CPU usage is being represented.

A unique combination of a metric name and a set of labels identifies a particular set of time-series data. This example uses a label *(It can have more labels)* called CPU to refer to usage of a specific CPU: node_cpu_seconds_total{cpu="0"}

### Metric Types
Metric types refer to different ways in which exporters represent the metric data they provide.

Metric types are not represented in any special way in a Prometheus server, but it is important to understand them in order to properly interpret your metrics.

- **Counter** : A counter is a single number that can only increase or be reset to zero. Counters represent cumulative values. Examples: Number of application restarts, Number of HTTP requests served by an application, etc. Example in querying: `node_cpu_seconds_total[5m]`
- **Gauge** : A gauge is a single number that can increase and decrease over time. Examples: CPU/Memory usage, current active threads, number of concurrent HTTP requests, etc. Example in querying: `node_memory_MemAvailable_bytes`
- **Histogram** : A histogram counts the number of observation/events that fall into a set of configurable buckets, each with its own separate time series. A histogram will use labels to differentiate between buckets. The below example provides the number of HTTP requests whose duration falls into each bucket. Histogram also include separate metric names to expose the *_sum* of all observed values and the total *_count* of events.
  - `prometheus_http_request_duration_seconds_bucket{le="0.3"}`
  - `prometheus_http_request_duration_seconds_bucket{le="1.0"}`
  - `prometheus_http_request_duration_seconds_sum`
  - `prometheus_http_request_duration_seconds_count`
- **Summary** : A summary is similar to a histogram, but it exposes metrics in the form of quantiles instead of buckets. While buckets divide values based on specific boundaries, quantiles divide values based on the percentiles into which they fall. Like histograms, summaries also expose the *_sum* and *_count* metrics. This value represents the number of HTTP requests whose duration falls within the 95th percentile of all requests or the top 5% longest requests: `prometheus_http_request_duration_seconds{quantile="0.95"}` or `go_gc_duration_seconds`

### Querying
Querying allows you to access and work with your metric data in Prometheus.

You can use PromQL (Prometheus Query Language) to write queries and retrieve useful information from the metric data collected by Prometheus.

You can use Prometheus queries in a variety of ways to obtain and work with data:
- Expression browser
- Prometheus HTTP API
- Visualization tools such as Grafana

#### Selectors


---
## Part 04: Visualization 


---
## Part 05: Collecting Metrics


---
## Part 06: Alerting


---
## Part 07: Advanced Concepts (HA and Federation)


---
## Part 08: Security


---
## Part 09: Client Libraries


---
## Part 10: References

1. [Official Prometheus Documentation](https://prometheus.io/docs/)
2. [Collection of Prometheus Alerting Rules](https://github.com/samber/awesome-prometheus-alerts)