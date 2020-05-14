---
title: Monitor Portworx with Sysdig
keywords: portworx, container, Kubernetes, storage, Docker, k8s, pv, persistent disk, monitoring, prometheus, alertmanager, servicemonitor, grafana, lighthouse, px-central, px-kvdb, sysdig
description: Learn how to monitor Portworx with Sysdig
---

Sysdig is a monitoring platform that allows you to monitor your Portworx cluster by running Sysdig agents on each node of your cluster.

![](/img/sysdigIntegration.png)

In order to use Sysdig with Portworx, you must deploy both products onto your Kubernetes node.

## Prerequisites

* You must have a Kubernetes cluster with Portworx installed on it

## Deploy Sysdig on Kubernetes with Portworx

1. Follow the [Kubernetes Agent Installation Steps](https://docs.sysdig.com/en/steps-for-kubernetes--vanilla-.html) in the Sysdig documentation to deploy the following YAML files:

    * sysdig-agent-clusterrole.yaml
    * sysdig-agent-configmap.yaml
    * sysdig-agent-daemonset-v2.yaml

2. Once you've deployed Sysdig, you must edit the `Sysdig-agent-configmap.yaml` file to enable Prometheus and configure the `px` processes to expose the metrics. Add the following fields to the `Sysdig-agent-configmap.yaml` file:

      ```text
      ...
      prometheus:
        enabled: true
        interval: 10
        process_filter:
          - include:
              process.cmdline: "px"
              conf:
                port: 9001
                path: "/metrics"
      ...
      ```

## Save and restore Dashboards

You can save and restore your Sysdig dashboards on your Portworx cluster. Follow the steps in the [Save and Restore Dashboards with Scripts](https://docs.sysdig.com/en/save-and-restore-dashboards-with-scripts.html) section of the Sysdig documentation to download the scripts and review instructions for running them.
