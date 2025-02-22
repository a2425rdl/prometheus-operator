---
weight: 100
toc: true
title: Design
menu:
    docs:
        parent: operator
lead: ""
images: []
draft: false
description: This document describes the design and interaction between the custom resource definitions that the Prometheus Operator introduces.
date: "2021-03-08T08:49:31+00:00"
---

This document describes the design and interaction between the custom resource definitions that the Prometheus Operator introduces.

The custom resources that the Prometheus Operator introduces are:

* [Prometheus](#prometheus)
* [Alertmanager](#alertmanager)
* [ThanosRuler](#thanosruler)
* [ServiceMonitor](#servicemonitor)
* [PodMonitor](#podmonitor)
* [Probe](#probe)
* [PrometheusRule](#prometheusrule)
* [AlertmanagerConfig](#alertmanagerconfig)

## Prometheus

The `Prometheus` custom resource definition (CRD) declaratively defines a desired Prometheus setup to run in a Kubernetes cluster. It provides options to configure replication, persistent storage, and Alertmanagers to which the deployed Prometheus instances send alerts to.

For each `Prometheus` resource, the Operator deploys a properly configured `StatefulSet` in the same namespace. The Prometheus `Pod`s are configured to mount a `Secret` called `<prometheus-name>` containing the configuration for Prometheus.

The CRD specifies which `ServiceMonitor`s should be covered by the deployed Prometheus instances based on label selection. The Operator then generates a configuration based on the included `ServiceMonitor`s and updates it in the `Secret` containing the configuration. It continuously does so for all changes that are made to `ServiceMonitor`s or the `Prometheus` resource itself.

If no selection of `ServiceMonitor`s is provided, the Operator leaves management of the `Secret` to the user, which allows to provide custom configurations while still benefiting from the Operator's capabilities of managing Prometheus setups.

## Alertmanager

The `Alertmanager` custom resource definition (CRD) declaratively defines a desired Alertmanager setup to run in a Kubernetes cluster. It provides options to configure replication and persistent storage.

For each `Alertmanager` resource, the Operator deploys a properly configured `StatefulSet` in the same namespace. The Alertmanager pods are configured to include a `Secret` called `<alertmanager-name>` which holds the used configuration file in the key `alertmanager.yaml`.

When there are two or more configured replicas the operator runs the Alertmanager instances in high availability mode.

## ThanosRuler

The `ThanosRuler` custom resource definition (CRD) declaratively defines a desired [Thanos Ruler](https://github.com/thanos-io/thanos/blob/main/docs/components/rule.md) setup to run in a Kubernetes cluster. With Thanos Ruler recording and alerting rules can be processed across multiple Prometheus instances.

A `ThanosRuler` instance requires at least one `queryEndpoint` which points to the location of Thanos Queriers or Prometheus instances. The `queryEndpoints` are used to configure the `--query` arguments(s) of the Thanos runtime.
Further information can also be found in the [Thanos doc](thanos.md).

## ServiceMonitor

The `ServiceMonitor` custom resource definition (CRD) allows to declaratively define how a dynamic set of services should be monitored. Which services are selected to be monitored with the desired configuration is defined using label selections. This allows an organization to introduce conventions around how metrics are exposed, and then following these conventions new services are automatically discovered, without the need to reconfigure the system.

For Prometheus to monitor any application within Kubernetes an `Endpoints` object needs to exist. `Endpoints` objects are essentially lists of IP addresses. Typically an `Endpoints` object is populated by a `Service` object. A `Service` object discovers `Pod`s by a label selector and adds those to the `Endpoints` object.

A `Service` may expose one or more service ports, which are backed by a list of multiple endpoints that point to a `Pod` in the common case. This is reflected in the respective `Endpoints` object as well.

The `ServiceMonitor` object introduced by the Prometheus Operator in turn discovers those `Endpoints` objects and configures Prometheus to monitor those `Pod`s.

The `endpoints` section of the `ServiceMonitorSpec`, is used to configure which ports of these `Endpoints` are going to be scraped for metrics, and with which parameters. For advanced use cases one may want to monitor ports of backing `Pod`s, which are not directly part of the service endpoints. Therefore when specifying an endpoint in the `endpoints` section, they are strictly used.

> Note: `endpoints` (lowercase) is the field in the `ServiceMonitor` CRD, while `Endpoints` (capitalized) is the Kubernetes object kind.

Both `ServiceMonitors` as well as discovered targets may come from any namespace. This is important to allow cross-namespace monitoring use cases, e.g. for meta-monitoring. Using the `ServiceMonitorNamespaceSelector` of the `PrometheusSpec`, one can restrict the namespaces `ServiceMonitor`s are selected from by the respective Prometheus server. Using the `namespaceSelector` of the `ServiceMonitorSpec`, one can restrict the namespaces the `Endpoints` objects are allowed to be discovered from.
To discover targets in all namespaces the `namespaceSelector` has to be empty:

```yaml
spec:
  namespaceSelector:
    any: true
```

## PodMonitor

The `PodMonitor` custom resource definition (CRD) allows to declaratively define how a dynamic set of pods should be monitored.
Which pods are selected to be monitored with the desired configuration is defined using label selections.
This allows an organization to introduce conventions around how metrics are exposed, and then following these conventions new pods are automatically discovered, without the need to reconfigure the system.

A `Pod` is a collection of one or more containers which can expose Prometheus metrics on a number of ports.

The `PodMonitor` object introduced by the Prometheus Operator discovers these pods and generates the relevant configuration for the Prometheus server in order to monitor them.

The `PodMetricsEndpoints` section of the `PodMonitorSpec`, is used to configure which ports of a pod are going to be scraped for metrics, and with which parameters.

Both `PodMonitors` as well as discovered targets may come from any namespace. This is important to allow cross-namespace monitoring use cases, e.g. for meta-monitoring.
Using the `namespaceSelector` of the `PodMonitorSpec`, one can restrict the namespaces the `Pods` are allowed to be discovered from.
To discover targets in all namespaces the `namespaceSelector` has to be empty:

```yaml
spec:
  namespaceSelector:
    any: true
```

## Probe

The `Probe` custom resource definition (CRD) allows to declarative define how groups of ingresses and static targets should be monitored. Besides the target, the `Probe` object requires a `prober` which is the service that monitors the target and provides metrics for Prometheus to scrape. This could be for example achieved using the [blackbox exporter](https://github.com/prometheus/blackbox_exporter/).

## PrometheusRule

The `PrometheusRule` custom resource definition (CRD) declaratively defines a desired Prometheus rule to be consumed by one or more Prometheus instances.

Alerts and recording rules can be saved and applied as YAML files, and dynamically loaded without requiring any restart.

## AlertmanagerConfig

The `AlertmanagerConfig` custom resource definition (CRD) declaratively specifies subsections of the Alertmanager configuration, allowing routing of alerts to custom receivers, and setting inhibit rules. The `AlertmanagerConfig` can be defined on a namespace level providing an aggregated config to Alertmanager. An example on how to use it is provided [here](../example/user-guides/alerting/alertmanager-config-example.yaml). Please be aware that this CRD is not stable yet.
