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
*replace xxx with your HEC token*

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

8. Open your Splunk instance and check for logs. Note they don't have span and trace metadata. This is because the current Java SDK version 1.x that ships with the Helm chart is not the latest one. In order to see span and trace metadata into logs you will have to add a properties file to your app (see Workaround #1) or upgrade the java sdk version to 2.0 (see Workaround #2).

## Workaround #1 (modify the app)

1. First, remove the app and install a new image with an application.properties. Also, add back the external IP address and the annotation.
```
helm uninstall springboot-starterkit-svc
helm install --values springboot-starterkit-appproperties-values.yaml springboot-starterkit-svc springboot/springboot-starterkit-svc
kubectl patch svc springboot-starterkit-svc -p '{"spec": {"type": "LoadBalancer", "externalIPs":["10.202.12.167"]}}'
kubectl patch deployment springboot-starterkit-svc -p '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-java":"splunk-otel/splunk-otel-collector"}}}}}'
```

2. Wait until all recently created springboot pods are ready
```
kubectl get pods
```

3. The app.properties file looks like this one here:
```
/src/main/resources/application.properties:

logging.pattern.console = %d{yyyy-MM-dd HH:mm:ss} - %logger{36} - %msg %logger{36} - %msg trace_id=%X{trace_id} span_id=%X{span_id} service=%X{service.name}, env=%X{deployment.environment} trace_flags=%X{trace_flags} %n %n
```
It is documented [here](https://docs.splunk.com/observability/en/gdi/get-data-in/application/java/instrumentation/connect-traces-logs.html#configure-your-logging-library)

4. Hit again the status endpoint to send logs to the Splunk instance
* http://10.202.12.167/status

5. Open your Splunk instance and check for logs. This time you will see logs coming through with trace_id and span_id.

## Workaround #2 (upgrade java sdk from 1.x to 2.x of the otel)

Java 2 automatically injects trace and span metadata into logs and no configuration (application.properties) is needed on the app side.

1. First, remove the current app with the application properties file and put back the previous app. Also include the external ip and the annotation.
```
helm uninstall springboot-starterkit-svc
helm install --values springboot-starterkit-values.yaml springboot-starterkit-svc springboot/springboot-starterkit-svc
kubectl patch svc springboot-starterkit-svc -p '{"spec": {"type": "LoadBalancer", "externalIPs":["10.202.12.167"]}}'
kubectl patch deployment springboot-starterkit-svc -p '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-java":"splunk-otel/splunk-otel-collector"}}}}}'
```

2. Wait until all recently created springboot pods are ready
```
kubectl get pods
```

3. Hit again the status endpoint to send logs to the Splunk instance
* http://10.202.12.167/status

4. Open your Splunk instance and check for logs. Make sure that the new logs don't have trace and space metadata.

5. Upgrade to Java instrumentation 2.0 using helm
```
helm upgrade splunk-otel-collector --values splunk_java2_values.yaml --set="splunkObservability.accessToken=***,clusterName=mycluster1,splunkObservability.realm=us1,gateway.enabled=false,splunkObservability.profilingEnabled=true,environment=lab,operator.enabled=true,certmanager.enabled=true,agent.discovery.enabled=true" splunk-otel-collector-chart/splunk-otel-collector --namespace splunk-otel --create-namespace
```
*replace *** with your o11y INGEST token*
*replace xxx with your HEC token*

6. Delete the app pods that are currently running. ReplicaSet will spin up new pods with the new java 2 injected by the operator.
```
kubectl delete pod springboot-starterkit-svc-df6567cb9-bjk9k springboot-starterkit-svc-df6567cb9-czv22 springboot-starterkit-svc-df6567cb9-jlsfr
```

7. Wait until all recently created springboot pods are ready
```
kubectl get pods
```

8. Hit again the status endpoint to send logs to the Splunk instance
* http://10.202.12.167/status

9. Open your Splunk instance and check for logs. This time you see trace and span metadata into the logs.

