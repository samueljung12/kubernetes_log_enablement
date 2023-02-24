# Sandbox Demo for Log Enablement!
Sandbox Demo showcasing Log Enablement w/ MiniKube

# Prerequisites
1. You must have Docker installed
2. You must have Kubectl installed
3. You must have Minikube installed 
4. It would be nice to have a Helm chart available for use but in case you don't and need one for reference, please refer here: https://github.com/DataDog/helm-charts/blob/main/charts/datadog/values.yaml 


# Step 1 - Enable Log Collection
To my understanding, there are 2 ways to go about setting your API key via the Helm Chart: 
1) You can set it in the command line: 
2) ```helm install <RELEASE_NAME> -f values.yaml --set datadog.site='datadoghq.com' --set datadog.apiKey=<API_KEY> --set datadog.appKey=<APP_KEY> datadog/datadog```

As you can see, you can also set the datadog.site and datadog.appKey as well! Pretty convenient eh?


# or

2) You can set via the Helm Chart:

For example:
``` 
datadog:
  apiKey: <Redacted>
  site: datadoghq.com #this defaults to datadoghq.com if not specified
  
  ...
  
  logs:
    enabled: true
    containerCollectAll: true
  ...
```

# Step 2 - Deploy Helm Chart
Now that we have enabled log collection, let's go ahead and deploy this bad boy. Please copy/paste the following command to deploy your Helm Chart:

```
helm install <RELEASE_NAME> -f values.yaml datadog/datadog
```
Here, a lot of people will get confused on what the heck this ```<RELEASE_NAME>``` is and what it should be set to. The answer is ANYTHING. You can set ```<RELEASE_NAME>``` to literally any lowercase string value (no upper case letters or numbers included please otherwise you'll encounter an error). It would be very helpful to remember this ```<RELEASE_NAME>``` and you'll see why soon ;)

# Step 3 - Verify your Helm Chart is deployed
Run the following command:
```
kubectl get pods
```
and this will return all the pods running in your current setup. We should be expecting to see 2 pods created (1 is the DD node agent while the other is the DD cluster agent). Your node pod name should take the format of ```<RELEASE_NAME>-datadog-xyz123``` (xyz123 will be different everytime you deploy your Helm Chart. this also applies to re-deploying the same file as well). Your cluster pod name should take the format of ```<RELEASE_NAME>-datadog-cluster-xyz123```.

# Step 4 - Verify logs are coming in your DD admin
You should see logs start populating in your Live Tail page if the above is configured correctly.
These logs are stdout/stderr or short for standard output & standard error meaning it is outputting whatever the container(s) in the pod are outputting. But now what if we want to collect from a custom log file? Can we do that? The answer is yes btw if you look at Step 5.

# Step 5 - Configure Helm Chart to collect logs from a file as opposed to stdout/stderr
We're gonna use our beloved integration redis as an example here and you can copy/paste the below configuration into a new file called ```redis.yaml```. 
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
 
Uncomment the annotations in your ```redis.yaml``` file:
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

Deploy your ```redis.yaml``` file with the following command:
```
kubectl apply -f redis.yaml
```

Add the following volumemounts and volumes in your Helm chart:
```
  volumes:
  - name: redis
    hostPath:
      path: /var/log
  volumeMounts:
  - mountPath: /var/log
    name: redis
```

Update your ```redis.yaml``` spec configuration as follows:
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
Re-deploy your Helm Chart using the helm upgrade command:
```
helm upgrade <RELEASE_NAME> -f values.yaml datadog/datadog
```

# Check the status
```
kubectl exec -it <AGENT_POD_NAME> agent status
```
# Delete your pod
```
kubectl delete pod <pod_name>
```
