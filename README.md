# Java auto instrumentation doesn't inject automatically span and trace metadata into logs

## Requirements
* K8s cluster [How to Install Kubernetes Cluster on Ubuntu 24.04 LTS](https://hbayraktar.medium.com/how-to-install-kubernetes-cluster-on-ubuntu-22-04-step-by-step-guide-7dbf7e8f5f99)
* A Splunk Enterprise or Cloud instance

## Installation steps

1. Install the patched springboot application using helm chart
```
helm install --values springboot-starterkit-values.yaml springboot-starterkit-svc springboot/springboot-starterkit-svc
```
We have patched the application to add a log emit inside the status controller. [See](https://github.com/josephrodriguez/springboot-starterkit/commit/8c506856bbb321f641ced6215a15f1067931de44)

2. Set the external IP address of the springboot app service
```
kubectl patch svc springboot-starterkit-svc -p '{"spec": {"type": "LoadBalancer", "externalIPs":["10.202.12.167"]}}'
```
*replace 10.202.12.167 with your own public ip address*

3. Now you should be able to hit the /status endpoint of the application
* http://10.202.12.167/status

4. Install the Otel collector and point the logs to your Splunk instance
```
helm install splunk-otel-collector --values splunk-values.yaml --set="splunkObservability.accessToken=***,clusterName=mycluster1,splunkObservability.realm=us1,gateway.enabled=false,splunkObservability.profilingEnabled=true,environment=lab,operator.enabled=true,certmanager.enabled=true,agent.discovery.enabled=true" splunk-otel-collector-chart/splunk-otel-collector --namespace splunk-otel --create-namespace
```
*replace *** with your o11y INGEST token*

5. Add the annotation to the springboot app deployment so the autoinstrumentation injects the java agent into the container
```
kubectl patch deployment springboot-starterkit-svc -p '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-java":"splunk-otel/splunk-otel-collector"}}}}}'
```

6. Wait until all pods spring boot pods gets recreated
```
kubectl get pods -A
```

7. Hit again the status endopoint to send logs to the Splunk instance
* http://10.202.12.167/status

8. Open your Splunk instance and check for logs. Note they don't have span and trace metadata. It is because the current java sdk version 1.x that ships with the helm chart is not the latest one. In order to see span and trace metadata into logs you will have to upgrade the java sdk version to 2.0.

## Workaround #1 (modify the app)

## Workaround #2 (upgrade java sdk from 1.x to 2.x of the otel)
