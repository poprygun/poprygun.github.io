---
layout: post
title:  "In case you want to use specific buildpack for your Tanzu workload..."
date:   2022-11-24 02:41:49 -0500
categories: tanzu tap buildpack clound-native-buildpacks
---

## Use Case

Tanzu workload requires specific buildpack or different version of the buildpack available in clusterstore.

## Context

Workload can be created specifying `clusterBuilder` parameter that can be configured with the required buildpack.

Below are the steps to configure a clusterbuilder.

### Load buildpack to clusterstore

```bash
kp clusterstore status default
kp clusterstore add default -b registry.tanzu.vmware.com/tanzu-python-buildpack/python:2.2.0 --dry-run
kp clusterstore add default -b registry.tanzu.vmware.com/tanzu-python-buildpack/python:2.2.0 --registry-verify-certs=false
```

### Create clusterbuilder

```bash
kp clusterbuilder create python-builder --tag harbor.h2o-4-1333.h2o.vmware.com/rpm/python-builder:2.2.0 --buildpack tanzu-buildpacks/python@2.2.0 
```

### Create workload specifying new clusterbuilder

```bash
tanzu apps workload create leave-behind \
--git-repo https://gitlab.eng.vmware.com/vmware-navigator-practice/tooling/leave-behind.git \
--type web --git-branch main --namespace default \
--env "STREAMLIT_SERVER_PORT=8080" \
--build-env "BP_CPYTHON_VERSION=3.10" \
--build-env "BP_POETRY_VERSION=1.2.2" \
--label app.kubernetes.io/part-of=leave-behind \
--label apps.tanzu.vmware.com/has-tests=true \
--annotation autoscaling.knative.dev/minScale=1 \
--namespace default \
--param clusterBuilder=python-builder \
--tail \
--yes
```
