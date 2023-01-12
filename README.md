# Sandbox Demo for Log Enablement!
Sandbox Demo showcasing Log Enablement w/ MiniKube

# Prerequisites
1. You must have Docker installed
2. You must have Kubectl installed
3. You must have Minikube installed 
4. It would be nice to have a Helm chart available for use but in case you don't and need one for reference, please refer here: https://github.com/DataDog/helm-charts/blob/main/charts/datadog/values.yaml 


# Step 1 - Enable Log Collection in your Helm Chart
For example:
``` 
datadog:
  apiKey: <Redacted>
  site: datadoghq.com
  
  ...
  
  logs:
    enabled: true
    containerCollectAll: true
  ...
```

# Step 2 - Deploy Helm Chart
```
helm install <RELEASE_NAME> -f values.yaml datadog/datadog
```
Or if you haven't set your API key and site variables in your Helm Chart, you can instead run 
```
helm install <RELEASE_NAME> -f values.yaml --set datadog.site='datadoghq.com' --set datadog.apiKey=<API_KEY> datadog/datadog
```

# Step 3 - Verify logs are indeed coming in the DD admin
You should see logs start populating in your Live Tail page if the above is configured correctly.
These logs are all stdout or stderr or short for standard output & standard error.

# [Optional] Step 4 - Configure Helm Chart to collect logs from a file as opposed to the stdout/stderr
We're gonna use redis as an example and you can copy/paste the below configuration into a files called redis.yaml. 
```
apiVersion: v1
kind: Pod
metadata:
  name: redis   
  # annotations:
  #   ad.datadoghq.com/redis.logs: |
  #     [{
  #         "type": "file",
  #         "path": "/var/log/alternatives.log",
  #         "source": "redis",
  #         "service": "redis"
  #     }]    
  labels:
    name: redis
spec:
  containers:
    - name: redis
      volumeMounts:
      - name: redis
        mountPath: /var/log
      image: redis
      ports:
        - containerPort: 6379
  volumes:
     - name: redis
       hostPath:
         path: /var/log
 ```




Uncomment the annotations in your Redis pod:
```
annotations:
    # ad.datadoghq.com/redis.logs: |
    #   [{
    #       "type": "file",
    #       "path": "/var/log/alternatives.log",
    #       "source": "redis",
    #       "service": "redis"
    #   }]    
```

Deploy your Redis pod
```
kubectl apply -f redis.yaml
```

Add the volumemounts and volumes in your Helm chart:
```
agents:
  volumes:
  - name: redis
    hostPath:
      path: /var/log
  volumeMounts:
  - mountPath: /var/log
    name: redis
```

Update your redis.yaml spec configuration as follows:
```
spec:
  containers:
    - name: redis
      volumeMounts:
      - name: redis
        mountPath: /var/log
      image: redis
      ports:
        - containerPort: 6379
  volumes:
     - name: redis
       hostPath:
         path: /var/log
```
Update your Helm Chart:
```
helm upgrade <RELEASE_NAME> -f values.yaml datadog/datadog
```

# Step 5 - Check the status
```
kubectl exec -it <AGENT_POD_NAME> agent status
```
