---
layout: post
title:  "In case you want to provision a Postgres operator on Tanzu Application Platform..."
date:   2022-11-25 02:41:49 -0500
categories: tanzu tap postgres operator
---

# Work In Progress...

## Use Case

Deploy Postgres instance as managed service on TAP.

## Context

Path of a least resistance for a RDBMS available to TAP workloads.

Below are the steps to configure a Postgres Operator.

## Install and configure Postgres operator

### Add target registry to Docker Desktop `insecure-reopsitories` configuration

```bash
docker login registry.tanzu.vmware.com
```

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "debug": true,
  "insecure-registries": ["harbor.target.registry.vmware.com"],
  "experimental": true
}
```

```bash
docker login harbor.target.registry.vmware.com
```

### Copy Tanzu TDS packages that include Postgres

```bash
imgpkg copy -b registry.tanzu.vmware.com/packages-for-vmware-tanzu-data-services/tds-packages:1.1.0 \
     --to-repo harbor.target.registry.vmware.com/rpm/tds-packages --registry-verify-certs='False'
```

### Define context

```bash
kubectl create ns postgres-databases
kubectl config set-context --current --namespace=postgres-databases
```

### Create secret

```bash
tanzu secret registry add data-services-repository-credentials \
    --username my-secert-username \
    --password my-secret-password \
    --server harbor.target.registry.vmware.com \
    --export-to-all-namespaces --yes
```

```bash
tanzu package repository add tanzu-data-services-repository --url harbor.target.registry.vmware.com/rpm/tds-packages:1.1.0 -n postgres-databases
```

### Verify that Postgres package is available

```bash
tanzu package available list -n postgres-databases
```

### Create or reuse the Storage Class with `VOLUMEBINDINGMODE=WaitForFirstConsumer`

### Inastall Operator package

```bash
tanzu package install postgres-operator --package-name postgres-operator.sql.tanzu.vmware.com --version 1.8.0 -n postgres-databases
```

### Verify installation

```bash
kubectl get serviceaccount
watch kubectl get all --selector app=postgres-operator 
kubectl logs -l app=postgres-operator
kubectl api-resources --api-group=sql.tanzu.vmware.com
```

## Create DB Instance

Define instance properties using postgres.yaml

```yaml
apiVersion: sql.tanzu.vmware.com/v1
kind: Postgres
metadata:
  name: postgres-sample
spec:
  memory: 800Mi
  cpu: 0.5
  storageClassName: postgres-storage-class
  monitorStorageClassName: postgres-storage-class
  storageSize: 2G
  pgConfig:
    dbname: postgres-sample
    username: admin
    appUser: superpassword
  highAvailability:
    enabled: false
```

Create a database instance

```bash
kubectl apply -f postgres.yaml -n postgres-databases
```

Verify instance creation

```bash
kubectl get postgres postgres-sample -o yaml
```

Monitor logs

```bash
kubectl logs pod/postgres-sample-0 --all-containers  -n postgres-databases
kubectl logs -l postgres-instance=postgres-sample --all-containers --max-log-requests 10 --tail=-1 --prefix
```

Verify instance access

```bash
kubectl exec -ti pod/postgres-sample -- pg_autoctl show state
```

Login to instance

```bash
kubectl exec -it postgres-sample-0 -- bash -c "psql"
```

### Accessing the instance credentials

```bash
dbname=$(kubectl get secret postgres-sample-db-secret -o go-template='{{.data.dbname | base64decode}}')
username=$(kubectl get secret postgres-sample-db-secret -o go-template='{{.data.username | base64decode}}')
password=$(kubectl get secret postgres-sample-db-secret -o go-template='{{.data.password | base64decode}}')
```

## Consume Instance from the application

### Application must be deployed in exploded form for buildpack to take effect

Add following section to `build.gradle`

`build.gradle`

```groovy
jar {
	enabled = false
}
```

### Add `service-ref` paramater to tanzu create command identifying to service to bind to `db=sql.tanzu.vmware.com/v1:Postgres:postgres-databases:postgres-sample`

```bash
tanzu apps workload create my-app \
--git-repo https://gitlab.eng.vmware.com/my-app.git \
--type web --git-branch main --namespace default \
--label "app.kubernetes.io/part-of=my-app" \
--build-env "BP_JVM_VERSION=17" \
--annotation "autoscaling.knative.dev/minScale=1" \
--env "JAVA_TOOL_OPTIONS=-Dmanagement.server.port=8080" \
--service-ref "db=sql.tanzu.vmware.com/v1:Postgres:postgres-databases:postgres-sample"
```

## Useful db access commands

```bash
kubectl exec -ti pod/postgres-sample-0 -- pg_autoctl show state
```

### Connect to Postgres container

```bash
kubectl exec -it postgres-sample-0 -- bash -c "psql"
```

### Extract credentials for access to the database from remote clients

```bash
host=$(kubectl get service postgres-sample -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
port=$(kubectl get service postgres-sample -o jsonpath='{.spec.ports[0].port}')
```

## Resources

Use [Service Bindings](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.3/tap/GUID-getting-started-consume-services.html#overview)

[Spring Cloud Bindings](https://github.com/vmware-tanzu/servicebinding) pickup Servivce Binding for Postgres and injects datasource variables

Make sure that [Services Toolkit is installed](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.3/tap/GUID-services-toolkit-install-services-toolkit.html)

Follow [steps to bind the application to Postgres instance](https://docs.vmware.com/en/VMware-Tanzu-SQL-with-Postgres-for-Kubernetes/1.7/tanzu-postgres-k8s/GUID-creating-service-bindings.html)

Folowing conventions are applied

[Spring Boot convention](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.3/tap/GUID-spring-boot-conventions-reference-CONVENTIONS.html#spring-boot-convention-1) is applied.  System `-D` properties are added to `JAVA_TOOL_OPTIONS` variable.  Used to configure Actuator management port for exapmple, as well as probes used to verify the health of the workload.

[Spring Boot Buildpack](https://github.com/paketo-buildpacks/spring-boot) conributes org.springframework.cloud:spring-cloud-bindings:1.10.0
[Service intent conventions](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.3/tap/GUID-spring-boot-conventions-reference-CONVENTIONS.html#service-intent-conventions-6) adds annotation and label to PodTemplateSpec for each dependency found - `postgresql` in our case marking the workload as a candidate for service binding
