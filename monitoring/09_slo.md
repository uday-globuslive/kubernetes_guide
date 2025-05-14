# SLIs, SLOs, and SLAs in Kubernetes Environments

## Introduction

Service Level Indicators (SLIs), Service Level Objectives (SLOs), and Service Level Agreements (SLAs) form the foundation of reliability engineering for modern applications. In Kubernetes environments, implementing a robust SLI/SLO framework helps teams deliver reliable services, focus on what matters most to users, and make data-driven decisions about platform and application reliability. This chapter explores how to define, measure, and implement SLIs and SLOs for Kubernetes workloads, with practical examples using Prometheus, Kubernetes monitoring tools, and custom solutions.

## Understanding SLIs, SLOs, and SLAs

### Service Level Indicators (SLIs)

SLIs are quantitative measures of service performance and reliability. They should directly reflect the user experience.

Common SLIs include:
- **Availability**: Percentage of successful requests 
- **Latency**: Distribution of request processing times
- **Throughput**: Rate of requests processed
- **Error Rate**: Percentage of failed requests
- **Durability**: Likelihood of data loss

### Service Level Objectives (SLOs)

SLOs are target values or ranges for SLIs that represent the desired service behavior.

Example SLOs:
- 99.9% of requests will complete successfully
- 95% of requests will be processed in under 200ms
- 99.99% durability of stored data

### Service Level Agreements (SLAs)

SLAs are contractual agreements with users that include consequences when service levels are not met.

- SLAs are typically less stringent than SLOs
- SLOs provide an "error budget" between the internal target and external promise
- Example: Your SLO might be 99.9% availability, while your SLA promises 99.5%

## SLI and SLO Framework for Kubernetes

### Selecting Meaningful SLIs

When selecting SLIs for Kubernetes environments, consider three key areas:

1. **Platform SLIs**: Health of the Kubernetes platform itself
2. **Infrastructure SLIs**: Performance of underlying resources
3. **Application SLIs**: End-user experience metrics

```
┌───────────────────────────────────────────────────┐
│                                                   │
│               Application SLIs                    │
│                                                   │
│  ┌───────────┐  ┌──────────┐  ┌────────────────┐  │
│  │ Latency   │  │ Errors   │  │ Throughput     │  │
│  └───────────┘  └──────────┘  └────────────────┘  │
│                                                   │
├───────────────────────────────────────────────────┤
│                                                   │
│               Platform SLIs                       │
│                                                   │
│  ┌───────────┐  ┌──────────┐  ┌────────────────┐  │
│  │ API Server│  │Scheduler │  │Control Plane   │  │
│  └───────────┘  └──────────┘  └────────────────┘  │
│                                                   │
├───────────────────────────────────────────────────┤
│                                                   │
│              Infrastructure SLIs                  │
│                                                   │
│  ┌───────────┐  ┌──────────┐  ┌────────────────┐  │
│  │ CPU       │  │ Memory   │  │ Network/Disk IO │  │
│  └───────────┘  └──────────┘  └────────────────┘  │
│                                                   │
└───────────────────────────────────────────────────┘
```

### Platform SLIs Examples

1. **Control Plane Availability**
   - API server availability
   - Controller manager availability
   - Scheduler availability

2. **Resource Management**
   - Node scheduling latency
   - Pod startup time
   - Resource allocation success rate

3. **Platform Operations**
   - Deployment success rate
   - Autoscaling response time
   - Node stability (node uptime)

### Application SLIs Examples

1. **Request-Based SLIs**
   - Request success rate
   - Request latency
   - Request throughput

2. **Processing SLIs**
   - Batch job completion rate
   - Queue processing time
   - Stream processing lag

3. **Storage SLIs**
   - Data durability
   - Read/write latency
   - Storage operation success rate

## Measuring SLIs in Kubernetes

### Prometheus-based SLI Measurement

Prometheus is the most common tool for measuring SLIs in Kubernetes environments.

#### 1. Availability SLI with Prometheus

```yaml
# Availability SLI using ratio of successful vs. total HTTP requests
- record: job:sli_availability:ratio_rate5m
  expr: |
    sum(rate(http_requests_total{status=~"2..|3.."}[5m])) by (job)
    /
    sum(rate(http_requests_total[5m])) by (job)
```

#### 2. Latency SLI with Prometheus

```yaml
# Latency SLI using histogram_quantile
- record: job:sli_latency_p95:histogram_quantile
  expr: |
    histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le))
```

#### 3. Control Plane SLI with Prometheus

```yaml
# API Server availability SLI
- record: kubernetes:apiserver_availability:ratio_rate5m
  expr: |
    sum(rate(apiserver_request_total{code=~"2.."}[5m]))
    /
    sum(rate(apiserver_request_total[5m]))
```

### Custom SLI Collection with Service Instrumentation

For applications that don't expose metrics directly, use client libraries to instrument code:

#### Example of Go Application Instrumentation

```go
package main

import (
    "time"
    "net/http"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"path", "method", "status"},
    )
    
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"path", "method", "status"},
    )
)

// Middleware to collect metrics
func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Custom response writer to capture status code
        rww := &responseWriterWrapper{ResponseWriter: w, statusCode: 200}
        
        // Call the next handler
        next.ServeHTTP(rww, r)
        
        // Record metrics
        duration := time.Since(start).Seconds()
        statusCode := rww.statusCode
        
        httpRequestsTotal.WithLabelValues(r.URL.Path, r.Method, string(statusCode)).Inc()
        httpRequestDuration.WithLabelValues(r.URL.Path, r.Method, string(statusCode)).Observe(duration)
    })
}

func main() {
    // Expose Prometheus metrics
    http.Handle("/metrics", promhttp.Handler())
    
    // Your application endpoints with metrics middleware
    http.Handle("/api/", metricsMiddleware(apiHandler()))
    
    http.ListenAndServe(":8080", nil)
}
```

### Black-box Monitoring with Synthetic Tests

Use synthetic tests to measure SLIs from an external perspective:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: blackbox-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: blackbox-exporter
  endpoints:
  - port: http
    interval: 30s
    scrapeTimeout: 10s
    metricRelabelings:
    - sourceLabels: [__address__]
      targetLabel: __param_target
    - sourceLabels: [__param_target]
      targetLabel: instance
    - targetLabel: __address__
      replacement: blackbox-exporter:9115
    relabelings:
    - sourceLabels: [__meta_kubernetes_service_name]
      targetLabel: service
    params:
      module:
      - http_2xx
    path: /probe
```

## Setting Appropriate SLOs

### SLO Development Process

1. **Identify critical user journeys**
2. **Select relevant SLIs for each journey**
3. **Determine appropriate targets based on:**
   - User expectations
   - Technical constraints
   - Business requirements
4. **Document SLOs formally**

### Practical SLO Examples for Kubernetes Workloads

#### API-based Application SLOs

```yaml
service: payment-api
slos:
  - name: availability
    description: "Successful HTTP requests"
    sli:
      metric: "sum(rate(http_requests_total{service='payment-api',status=~'2..|3..'}[5m])) / sum(rate(http_requests_total{service='payment-api'}[5m]))"
    target: 99.95%
    window: 30d
    
  - name: latency
    description: "Request processing time"
    sli:
      metric: "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service='payment-api'}[5m])) by (le))"
    target: 200ms
    window: 30d
```

#### Control Plane SLOs

```yaml
service: kubernetes-control-plane
slos:
  - name: api-server-availability
    description: "API server availability"
    sli:
      metric: "sum(rate(apiserver_request_total{code=~'2..'}[5m])) / sum(rate(apiserver_request_total[5m]))"
    target: 99.99%
    window: 30d
    
  - name: scheduling-latency
    description: "Time to schedule a pod"
    sli:
      metric: "histogram_quantile(0.95, sum(rate(scheduler_scheduling_algorithm_duration_seconds_bucket[5m])) by (le))"
    target: 1s
    window: 30d
```

#### Database SLOs

```yaml
service: postgres-database
slos:
  - name: availability
    description: "Database availability"
    sli:
      metric: "avg_over_time(pg_up[5m])"
    target: 99.99%
    window: 30d
    
  - name: read-latency
    description: "95th percentile query latency for read operations"
    sli:
      metric: "histogram_quantile(0.95, sum(rate(pg_stat_activity_max_tx_duration{state='active',query=~'SELECT.*'}[5m])) by (le))"
    target: 50ms
    window: 30d
```

## Implementing Error Budgets

Error budgets quantify how much service downtime or degradation is acceptable within an SLO period.

### Error Budget Calculation

For an availability SLO of 99.9% over 30 days:

```
Total minutes in 30 days = 30 * 24 * 60 = 43,200 minutes
Error budget (0.1%) = 43,200 * 0.001 = 43.2 minutes
```

This means your service can be unavailable for up to 43.2 minutes in a 30-day period while still meeting the SLO.

### Error Budget Implementation with Prometheus

```yaml
# Record current error budget consumption
- record: slo:error_budget_consumption:ratio
  expr: |
    1 - (
      (
        # Current availability over the SLO window
        avg_over_time(job:sli_availability:ratio_rate5m[30d])
      )
      /
      # SLO target (e.g., 0.999 for 99.9%)
      (0.999)
    )
```

### Error Budget Alerting

```yaml
# Alert when error budget is being consumed too quickly
- alert: ErrorBudgetBurnRateTooHigh
  expr: |
    # Predicting we'll consume 100% of error budget before the window ends
    (
      # Target error budget (1 - SLO)
      (1 - 0.999) 
      -
      # Current availability over the window
      (1 - avg_over_time(job:sli_availability:ratio_rate5m[30d]))
    )
    # Days into the window
    / 30
    # Project to the full window length
    * 100 < 0
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Error budget consumption is too high"
    description: "Service is consuming error budget too fast and will likely miss SLO target for this window"
```

## Visualizing SLOs in Kubernetes Dashboards

### Grafana Dashboard for SLO Tracking

```json
{
  "dashboard": {
    "id": null,
    "title": "Service SLO Dashboard",
    "tags": ["slo", "kubernetes"],
    "timezone": "browser",
    "editable": true,
    "panels": [
      {
        "title": "Availability SLO",
        "type": "gauge",
        "gridPos": {
          "h": 8,
          "w": 8,
          "x": 0,
          "y": 0
        },
        "options": {
          "minValue": 0,
          "maxValue": 100,
          "thresholds": [
            {
              "color": "red",
              "value": 0
            },
            {
              "color": "yellow",
              "value": 99.5
            },
            {
              "color": "green",
              "value": 99.9
            }
          ]
        },
        "targets": [
          {
            "expr": "100 * avg_over_time(job:sli_availability:ratio_rate5m[30d])",
            "format": "time_series"
          }
        ]
      },
      {
        "title": "Error Budget Remaining",
        "type": "gauge",
        "gridPos": {
          "h": 8,
          "w": 8,
          "x": 8,
          "y": 0
        },
        "options": {
          "minValue": 0,
          "maxValue": 100,
          "thresholds": [
            {
              "color": "red",
              "value": 0
            },
            {
              "color": "yellow",
              "value": 20
            },
            {
              "color": "green",
              "value": 50
            }
          ]
        },
        "targets": [
          {
            "expr": "100 * (1 - slo:error_budget_consumption:ratio)",
            "format": "time_series"
          }
        ]
      },
      {
        "title": "Latency SLO (95th Percentile)",
        "type": "gauge",
        "gridPos": {
          "h": 8,
          "w": 8,
          "x": 16,
          "y": 0
        },
        "options": {
          "minValue": 0,
          "maxValue": 500,
          "thresholds": [
            {
              "color": "green",
              "value": 0
            },
            {
              "color": "yellow",
              "value": 150
            },
            {
              "color": "red",
              "value": 200
            }
          ],
          "unit": "ms"
        },
        "targets": [
          {
            "expr": "1000 * job:sli_latency_p95:histogram_quantile",
            "format": "time_series"
          }
        ]
      },
      {
        "title": "Availability Over Time",
        "type": "graph",
        "gridPos": {
          "h": 9,
          "w": 24,
          "x": 0,
          "y": 8
        },
        "targets": [
          {
            "expr": "100 * job:sli_availability:ratio_rate5m",
            "format": "time_series",
            "legendFormat": "Availability"
          },
          {
            "expr": "100 * 0.999",
            "format": "time_series",
            "legendFormat": "SLO Target (99.9%)"
          }
        ],
        "yaxes": [
          {
            "format": "percent",
            "min": 98,
            "max": 100
          },
          {
            "format": "short",
            "show": false
          }
        ]
      }
    ]
  }
}
```

## SLO Implementation with Service Mesh

Service meshes like Istio provide built-in capabilities for measuring SLIs and implementing SLOs.

### Istio Metrics for SLI Measurement

```yaml
# Using Istio metrics for SLI calculation
- record: service:availability:ratio_rate5m
  expr: |
    sum(rate(istio_requests_total{response_code=~"2..|3..",destination_service="my-service.default.svc.cluster.local"}[5m]))
    /
    sum(rate(istio_requests_total{destination_service="my-service.default.svc.cluster.local"}[5m]))

- record: service:latency:p95
  expr: |
    histogram_quantile(0.95, sum(rate(istio_request_duration_milliseconds_bucket{destination_service="my-service.default.svc.cluster.local"}[5m])) by (le))
```

### Implementing SLOs with Istio

1. **Service Level Objective Configuration**

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: ServiceLevelObjective
metadata:
  name: availability-slo
  namespace: default
spec:
  service: my-service
  goal: 0.999
  description: "99.9% availability for my-service"
  period: 30d
  indicator:
    metric: service:availability:ratio_rate5m
    type: METRIC
```

2. **Istio Virtual Service with Reliability Features**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service
  namespace: default
spec:
  hosts:
  - my-service
  http:
  - route:
    - destination:
        host: my-service
        subset: v1
      weight: 90
    - destination:
        host: my-service
        subset: v2
      weight: 10
    timeout: 0.5s
    retries:
      attempts: 3
      perTryTimeout: 0.2s
      retryOn: gateway-error,connect-failure,refused-stream
```

## Advanced SLO Patterns

### Multi-Window SLOs

Multi-window SLOs evaluate reliability over both short and long timeframes:

```yaml
service: payment-api
slos:
  - name: availability
    description: "Successful HTTP requests"
    sli:
      metric: "sum(rate(http_requests_total{service='payment-api',status=~'2..|3..'}[5m])) / sum(rate(http_requests_total{service='payment-api'}[5m]))"
    target: 99.9%
    windows:
      - duration: 24h
        allowed_violations: 0
      - duration: 30d
        allowed_violations: 3
```

### SLOs for Different Service Tiers

Different service tiers may have different reliability requirements:

```yaml
service: data-processing
tiers:
  - name: critical
    slos:
      - name: processing-success
        target: 99.99%
        window: 30d
      - name: processing-latency
        target: 10s
        window: 30d
  
  - name: batch
    slos:
      - name: processing-success
        target: 99.9%
        window: 30d
      - name: processing-latency
        target: 300s
        window: 30d
```

### Seasonality-Aware SLOs

For services with varying traffic patterns:

```yaml
service: retail-website
slos:
  - name: availability
    description: "Successful HTTP requests"
    target: 99.95%
    window: 30d
    adjustments:
      - condition: "month() == 11 || month() == 12"  # Holiday season
        target: 99.99%
```

## Converting SLOs to Actionable Engineering Work

### SLO-Based Prioritization Framework

Use SLOs to prioritize reliability work:

1. **Error budget-based prioritization**
   - When significantly out of SLO: Focus 100% on reliability
   - When close to SLO boundary: Balance reliability and features
   - When comfortably within SLO: Focus on new features

2. **Sample decision matrix**

```
| Error Budget Status         | Engineering Focus                          |
|-----------------------------|-------------------------------------------|
| >75% consumed               | 100% reliability, 0% features              |
| 50-75% consumed             | 75% reliability, 25% features              |
| 25-50% consumed             | 50% reliability, 50% features              |
| <25% consumed               | 25% reliability, 75% features              |
```

### Infrastructure Improvements Based on SLO Data

When specific SLOs are at risk:

1. **Latency SLO issues:**
   - Implement caching
   - Optimize database queries
   - Scale horizontally
   - Use CDNs for static content

2. **Availability SLO issues:**
   - Implement circuit breakers
   - Add redundancy
   - Improve auto-healing
   - Enhance monitoring and alerting

## Implementing SLAs for External Customers

### From SLOs to SLAs

When creating SLAs for customers:

1. **Set SLA targets looser than internal SLOs**
   - Example: Internal SLO = 99.95%, External SLA = 99.9%
   - This provides a safety buffer for unexpected issues

2. **Define clear measurement methodology**
   - How availability is calculated
   - What counts as downtime
   - Exclusions (e.g., planned maintenance)

3. **Specify appropriate remediation**
   - Service credits
   - Financial penalties
   - Termination rights

### Sample SLA Document for Kubernetes-Based Service

```markdown
# Service Level Agreement for MyService

## Service Commitment

MyCompany commits to a Monthly Uptime Percentage of at least 99.9% for MyService.

## Definitions

- "Monthly Uptime Percentage" is calculated by subtracting from 100% the percentage of minutes during the month in which MyService was Unavailable.
- "Unavailable" means that all requests to the service API fail with 5xx errors or timeout.
- "Exclusions" include:
  - Factors outside our reasonable control
  - Scheduled maintenance (announced at least 48 hours in advance)
  - Beta or preview features

## Service Credits

If the Monthly Uptime Percentage falls below 99.9%, the customer is eligible for the following Service Credits:
- <99.9% but ≥99.0%: 10% of monthly service fee
- <99.0% but ≥95.0%: 25% of monthly service fee
- <95.0%: 100% of monthly service fee

## Credit Process

To receive Service Credits, customer must submit a claim within 30 days of the incident, including:
- "SLA Credit Request" in the subject line
- Dates and times of the Unavailability
- Affected resources or operations
- Request logs documenting the errors (if available)
```

## Tools for SLO Management

### Open Source SLO Tools

1. **Sloth**
   - Kubernetes-native SLO framework
   - Generates Prometheus recording and alerting rules from SLO definitions

```yaml
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: payment-service-slo
  namespace: monitoring
spec:
  service: "payment-service"
  labels:
    owner: "platform-team"
    tier: "1"
  slos:
    - name: "availability"
      objective: 99.9
      description: "Availability of payment processing service"
      sli:
        events:
          errorQuery: sum(rate(http_requests_total{service="payment",code=~"5.."}[{{.window}}]))
          totalQuery: sum(rate(http_requests_total{service="payment"}[{{.window}}]))
      alerting:
        page:
          enabled: true
          alertName: "PaymentServiceHighErrorRate"
          labels:
            severity: "critical"
            team: "payments"
          annotations:
            summary: "High error rate on payment service"
            description: "Payment service has high error rate"
```

2. **OpenSLO**
   - YAML specification for defining SLOs
   - Multi-vendor support

```yaml
apiVersion: openslo/v1
kind: SLO
metadata:
  name: api-availability
  displayName: API Availability
spec:
  service: payment-api
  description: Availability SLO for the payment API
  indicator:
    ratio:
      errors:
        query:
          prometheus:
            query: sum(rate(http_requests_total{status=~"5..",service="payment-api"}[{{.window}}]))
      total:
        query:
          prometheus:
            query: sum(rate(http_requests_total{service="payment-api"}[{{.window}}]))
  objectives:
    - displayName: 30 Day Availability
      target: 0.999
      timeWindow:
        calendar:
          duration: 30d
  alertPolicies:
    - name: budget-burn-fast
      burn-rate:
        threshold: 14.4
        interval: 1h
```

3. **Pyrra**
   - Kubernetes-native SLO platform
   - UI for visualizing SLOs and error budgets

```yaml
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: http-availability
  namespace: monitoring
spec:
  target: 0.999
  description: "HTTP request availability"
  window: 30d
  service: payment-service
  indicator:
    ratio:
      errors:
        metric: http_requests_total{code=~"5.."}
      total:
        metric: http_requests_total
```

### Commercial SLO Platforms

Several commercial platforms provide SLO management capabilities:

1. **Datadog SLO Management**
   - Built-in SLO tracking
   - Error budget monitoring
   - Customizable dashboards and alerts

2. **Dynatrace SLO Management**
   - AI-powered SLO insights
   - Automatic baseline detection
   - Real-time error budget tracking

3. **New Relic SLO Management**
   - Multi-service SLO dashboards
   - Error budget alerting
   - Integration with incident management

## Implementing an SLO Culture

### SLO Process Implementation

1. **SLO Review Process**
   - Regular review meetings (bi-weekly or monthly)
   - Stakeholder involvement (product, engineering, ops)
   - Adjustment of SLOs based on business needs and technical capabilities

2. **SLO Documentation Template**

```yaml
service: payment-processing
owner: payments-team
description: "Processes all payment transactions"
dependencies:
  - database-cluster
  - auth-service
slos:
  - name: availability
    description: "Successful payment processing requests"
    target: 99.95%
    window: 30d
    measurement:
      sli: 
        numerator: "successful_requests"
        denominator: "total_requests"
      implementation:
        prometheus:
          query: "sum(rate(payment_requests_total{status='success'}[5m])) / sum(rate(payment_requests_total[5m]))"
    alerting:
      budget_burn_rates:
        - window: 1h
          burn_rate: 14.4  # will consume 100% of budget in 1/14.4 of time window
          alert:
            name: FastBurn
            message: "Error budget burning too fast"
    reported_to:
      - platform_team
      - leadership_dashboard
```

### SLO Culture Best Practices

1. **Focus on customer impact**
   - SLOs should reflect actual user experience
   - Prioritize user-facing metrics over internal ones

2. **Use SLOs for decision making**
   - Base deployment frequency on error budget
   - Prioritize reliability work based on SLO compliance
   - Make architectural decisions with SLOs in mind

3. **Iterative improvement**
   - Start simple and refine over time
   - Begin with key services and expand coverage
   - Adjust targets based on real-world data

## Case Study: Implementing SLOs for a Kubernetes E-Commerce Platform

### Initial SLO Implementation

An e-commerce company running on Kubernetes implemented the following SLOs:

```yaml
# User-facing services
service: product-catalog
slos:
  - name: availability
    target: 99.95%
    window: 30d
  - name: latency
    target: 200ms at 95th percentile
    window: 30d

service: checkout
slos:
  - name: availability
    target: 99.99%
    window: 30d
  - name: latency
    target: 500ms at 95th percentile
    window: 30d

# Infrastructure services
service: kubernetes-control-plane
slos:
  - name: api-availability
    target: 99.99%
    window: 30d

service: database-cluster
slos:
  - name: availability
    target: 99.995%
    window: 30d
  - name: latency
    target: 50ms at 95th percentile
    window: 30d
```

### Results and Learnings

After implementing SLOs:

1. **Reliability improvements**
   - 78% reduction in customer-impacting incidents
   - Better prioritization of infrastructure work

2. **Engineering culture improvements**
   - Data-driven reliability discussions
   - Clearer communication between teams
   - Better alignment between business and technical priorities

3. **Key learnings**
   - Start with fewer, more meaningful SLOs
   - Ensure alignment between teams on SLO definitions
   - Regularly review and adjust targets
   - Balance time spent on SLO implementation vs. improvements

## Conclusion

Implementing SLIs, SLOs, and SLAs in Kubernetes environments creates a structured approach to reliability engineering. By defining meaningful metrics, setting appropriate targets, and building a culture around reliability objectives, teams can deliver more reliable services while focusing engineering efforts where they matter most. Remember that SLOs are a journey, not a destination—start simple, iterate, and continuously improve based on real-world experience and changing business requirements.

## References

- [Google SRE Book: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Kubernetes SLOs with Prometheus](https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/#toc-14)
- [SLO Adoption: A Journey Not a Destination](https://landing.google.com/sre/resources/practicesandprocesses/slo-adoption-case-studies/)
- [Pyrra: Kubernetes SLO Infrastructure](https://github.com/pyrra-dev/pyrra)
- [Sloth: Kubernetes SLO Framework](https://github.com/slok/sloth)
- [OpenSLO Specification](https://github.com/OpenSLO/OpenSLO)
- [Reliability Toolkit for Kubernetes](https://cloud.google.com/blog/products/containers-kubernetes/introducing-gke-reliability-toolkit)