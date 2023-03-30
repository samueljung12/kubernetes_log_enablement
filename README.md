# Sandbox Demo for Log Enablement!
Sandbox Demo showcasing Log Enablement w/ MiniKube

# Prerequisites
1. You must have Docker installed
2. You must have Kubectl installed
3. You must have Minikube installed 
4. It would be nice to have a Helm chart available for use but in case you don't and need one for reference, please refer here: https://github.com/DataDog/helm-charts/blob/main/charts/datadog/values.yaml 

KB for installing everything: https://datadoghq.atlassian.net/wiki/spaces/TS/pages/1248530082/How+to+test+Kubernetes+yourself#Setup-Minikube-environment

# Step 1 - Enable Log Collection
To my understanding, there are 2 ways to go about setting your API key via the Helm Chart: 
1) You can set it in the command line: 
```helm install <RELEASE_NAME> -f values.yaml --set datadog.site='datadoghq.com' --set datadog.apiKey=<API_KEY> --set datadog.appKey=<APP_KEY> datadog/datadog```

As you can see, you can also set the ```datadog.site``` and ```datadog.appKey``` as well!


# or

2) You can set via the Helm Chart:

For example:
``` 
datadog:
  apiKey: <API_KEY>
  site: datadoghq.com
  clusterName: <CLUSTER_NAME> 
  # Disable kubelet TLS Verification in minikube
  kubelet:
    tlsVerify: false
  # confd: {}
  kubeStateMetricsEnabled: false
  kubeStateMetricsCore:
    enabled: false
  logs:
    enabled: true
    containerCollectAll: true
  apm:
    socketEnabled: false
    portEnabled: false
  processAgent:
    enabled: true
    processCollection: true
  orchestratorExplorer:
    enabled: true
  
  # These 3 integrations error by default in minikube
  ignoreAutoConfig:
    - etcd
    - kube_controller_manager 
    - kube_scheduler 
clusterAgent:
  enabled: true
  admissionController:
    enabled: false
    mutateUnlabelled: false
```

# Step 2 - Deploy Helm Chart
Now that we have enabled log collection, let's go ahead and deploy this. Please copy/paste the following command to deploy your Helm Chart:

```
helm install <RELEASE_NAME> -f values.yaml datadog/datadog
```
Here, a lot of people will get confused on what this ```<RELEASE_NAME>``` is and what it should be set to. The answer is anything! You can set ```<RELEASE_NAME>``` to literally any lowercase string value (no upper case letters or numbers included please otherwise you'll encounter an error). It would be very helpful to remember this ```<RELEASE_NAME>``` and you'll see why soon.

# Step 3 - Verify your Helm Chart is deployed
Run the following command:
```
kubectl get pods
```
and this will return all the pods running in your current setup. We should be expecting to see 2 pods created (1 is the DD node agent while the other is the DD cluster agent). Your node pod name should take the format of ```<RELEASE_NAME>-datadog-xyz123``` (xyz123 will be different everytime you deploy your Helm Chart. this also applies to re-deploying the same file as well). Your cluster pod name should take the format of ```<RELEASE_NAME>-datadog-cluster-agent-xyz123-xyz123```.

# Step 4 - Verify logs are coming in your DD admin
You should see logs start populating in your Live Tail page if the above is configured correctly.
These logs are stdout/stderr or short for standard output & standard error meaning it is outputting whatever the container(s) in the pod are outputting. But now what if we want to collect from a custom log file?

# Step 5 - Configure Helm Chart to collect logs from a custom file as opposed to stdout/stderr
We're gonna use a common integration called redis as an example here and you can copy/paste the below configuration into a new file called ```redis.yaml```. 
```
apiVersion: v1
kind: Pod
metadata:
  name: redis
  annotations:
    ad.datadoghq.com/redis.logs: '[{"source": "redis","service": "sam_redis","tags": ["env:sam_prod"]}]'
  labels:
    name: redis
spec:
  containers:
    - name: redis
      image: redis:latest
      ports:
        - containerPort: 6379
 ```
 
Let's delete the current redis pod:
```
kubectl delete pod redis
```

Please copy/paste the below into your ```redis.yaml``` file:
```
apiVersion: v1
kind: Pod
metadata:
  name: redis
  annotations:
    ad.datadoghq.com/redis.logs: |
      [{
          "type": "file",
          "path": "/var/log/example/app.log",
          "source": "redis",
          "service": "sam_redis"
      }]
spec:
  containers:
    - name: redis
      image: redis
      command: [ "/bin/sh", "-c", "--" ]
      args: [ "while true; do sleep 1; echo `date` example file log >> /var/log/example/app.log; done;" ]
      volumeMounts:
      - name: redis
        mountPath: /var/log/example
  volumes:
     - name: redis
       hostPath:
         path: /var/log/example 
```

Deploy your ```redis.yaml``` file with the following command:
```
kubectl apply -f redis.yaml
```

Add the following volumemounts and volumes in your Helm chart:
```
 agents:
  volumes:
  - name: redis
    hostPath:
      path: /var/log/example
  volumeMounts:
  - mountPath: /var/log/example
    name: redis
```

Re-deploy your Helm Chart using the helm upgrade command:
```
helm upgrade <RELEASE_NAME> -f values.yaml datadog/datadog
```

# Check the status
```
kubectl exec -it <AGENT_POD_NAME> agent status
```
# Get the list of your pods
```
kubectl get pods
```
# Delete your pod
```
kubectl delete pod <POD_NAME>
```
Collaboration KBs:
1) https://datadoghq.atlassian.net/wiki/spaces/TS/pages/1248530082/How+to+test+Kubernetes+yourself by Steve Wenzel (recommended read)
