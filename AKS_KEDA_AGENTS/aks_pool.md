First create the docker registry secrets using below command 

```
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=sivabalaji1995 \
  --docker-password=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 \
  --docker-email=sivanimmakayala07@gmail.com
```

Once it is created , create a secret object with the values of azure devops organisation url, pat token with all the necessaru permissions and agent pool name. 

```secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: azp-secret
type: Opaque
stringData:
  AZP_URL: https://dev.azure.com/sivabalaji280 
  AZP_TOKEN: 56BKNGHIvMnvcW83zYdpABZlLN2gowtJQ47TR0UVexDqEFW0JfA0JQQJ99CDACAAAAAAAAAAAAASAZDO45Eq 
  AZP_POOL: AKS-AGENT-POOL
type: Opaque

```

Once the secret is create, you have to create the deployment with minimum replicas as 1 by injecting all the values as env variables to the deploument. 

```deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-agent-pool
  labels:
    app: aks-agent-pool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aks-agent-pool
  template:
    metadata:
      labels:
        app: aks-agent-pool
    spec:
      imagePullSecrets:
      - name: ghcr-secret
      containers:
      - name: aks-agent-pool
        image: ghcr.io/sivabalaji1995/azp-java-agent:latest
        imagePullPolicy: Always
        env:
        - name: AZP_URL
          valueFrom:
            secretKeyRef:
              name: azp-secret
              key: AZP_URL
        - name: AZP_TOKEN
          valueFrom:
            secretKeyRef:
              name: azp-secret
              key: AZP_TOKEN
        - name: AZP_POOL
          valueFrom:
            secretKeyRef:
              name: azp-secret
              key: AZP_POOL
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "250m"

```

Once the deploument is created, a replica is created and it can be seen in the azure devops portal.


![alt text](<images/image copy.png>)

Once the deploument is creayed, you can priceed creaying the trigger authentication and scaledobject. 
- Thrigger authentication is to tell KEDA to tell where the secret is and what is the target and how to authenticate it.
- Scaled object is to tell the KEDA what is the threshold value and based on that to scale the pods. it internally created the HPA object.

```triggerauthentication.yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azp-trigger-auth
spec:
  secretTargetRef:
    - parameter: personalAccessToken
      name: azp-secret
      key: AZP_TOKEN
```

```scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: aks-agent-pool-scaledobject
spec:
  scaleTargetRef: 
    name: aks-agent-pool
  minReplicaCount: 1
  maxReplicaCount: 5
  pollingInterval: 15
  cooldownPeriod: 30
  triggers:
  - type: azure-pipelines
    metadata:
      organizationURL: https://dev.azure.com/sivabalaji280
      poolName: AKS-AGENT-POOL
      targetPendingBuilds: "1"
    authenticationRef:
      name: azp-trigger-auth
    
```

once these are deployed you can check the HPA status of the min and maximum pods.

By running more pipelines, keda scale the pods and agents. 

One blocker is , it is not possible to scaledown the pods to zero in azure devops as inorder the jobs to be in queue, it should have atleast one agent up and running insie the azure devops agent pool. 

You can find the images in the github image repository. 

The final result is 

![alt text](images/agentpool.png)