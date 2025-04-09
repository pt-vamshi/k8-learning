## Step 1: Pod Basics
- Read about the Pod definition, lifecycle, and uses from the documentation.
- Create a Pod named basic-pod running nginx using:
### Imperative method (Using command):
 ```kubectl run basic-pod --image=nginx --restart=Never```

### Declarative method (YAML manifest):
```
apiVersion: v1
kind: Pod
metadata:
  name: basic-pod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

## Step 2. Multi-Container Pods
- Understand the concept of multi-container Pods and how they share networking and storage.
- Create a Pod named multi-pod with:
  - nginx container exposing port 80.
  - busybox container running sleep 3600.
  - Use a shared emptyDir volume to exchange data between containers.

### Imperative Approach:
``` kubectl run ecommerce-pod --image=nginx --restart=Never --port=80 ```

</br><b>Note:</b> Add second container by editing the pod yaml file and applying the changes

### Declarative Approach:

Create the following YAML file ecommerce-pod.yaml:
```
apiVersion: v1
kind: Pod
metadata:
  name: ecommerce-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
```     

### Deploy the Pod
Run the following command to deploy the pod:
``` kubectl apply -f nginx-pod.yaml ```

### Verify the Pod Status
``` kubectl get pods ```

### Describe the Pod
``` kubectl describe pod ecommerce-pod ```

### Access the Pod Logs
View the logs of the NGINX Pod to ensure it’s running correctly:
``` kubectl logs ecommerce-pod ```


## Step 3. Add Init Containers
### Add an init container to ensure the nginx container starts only after necessary dependencies are initialized.
YAML Update:
```
initContainers:
- name: init-busybox
  image: busybox
  command: ["sh", "-c", "echo Initializing frontend; sleep 5"]
```

## Step 4. Secure the Pod
### Add securityContext features to the Pod:
  - Ensure all containers run as non-root users.
  - Drop unnecessary Linux capabilities.
YAML Update:
```
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: nginx
    image: nginx
    securityContext:
      capabilities:
        drop:
        - ALL
  - name: busybox
    image: busybox
    securityContext:
      capabilities:
        drop:
        - ALL
```

### Securing Pods with Service Accounts
Pods can inherit permissions through Service Accounts. Let’s secure the Pod by using a Service Account for access management.

### Create a Service Account
Define the service account in your YAML file:
```
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: nginx-sa
```

### Bind the Service Account to the Pod
Update the Pod definition to use the created Service Account:

```
apiVersion: v1  
kind: Pod  
metadata:  
  name: nginx-pod  
spec:  
  serviceAccountName: nginx-sa  
  containers:  
    - name: nginx  
      image: nginx:latest  
      ports:  
        - containerPort: 80
```

### Deploy the Pod with the Service Account
Re-deploy the Pod with the new configuration:
``` kubectl apply -f nginx-pod-sa.yaml  ```

### Verify the Service Account Binding
Run the following to check if the Service Account is properly assigned:
``` kubectl get pod nginx-pod -o=jsonpath='{.spec.serviceAccountName}'  ```

### Implementing RBAC for Pod Access
Control who can access the resources in your Pods using Kubernetes RBAC. In this task, you’ll configure RBAC to allow only authorized users to access the Pod.

Define a Role
Create a Role in a specific namespace (e.g., default) that allows reading Pod details:
```
apiVersion: rbac.authorization.k8s.io/v1  
kind: Role  
metadata:  
  namespace: default  
  name: pod-reader  
rules:  
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

Bind the Role to a User or Service Account
Define a RoleBinding to associate the user/service account with the Role:
```
apiVersion: rbac.authorization.k8s.io/v1  
kind: RoleBinding  
metadata:  
  name: pod-reader-binding  
  namespace: default  
subjects:  
- kind: ServiceAccount  
  name: nginx-sa  
  namespace: default  
roleRef:  
  kind: Role  
  name: pod-reader  
  apiGroup: rbac.authorization.k8s.io
```

Apply the Role and RoleBinding
Run the following commands to apply the configurations:
```
kubectl apply -f role.yaml  
kubectl apply -f rolebinding.yaml  
```

Step 5. Configure Resource Limits
Configure resource requests and limits for the containers:
nginx: Request 100m CPU and 128Mi memory; Limit 250m CPU and 256Mi memory.
busybox: Request 50m CPU and 64Mi memory; Limit 100m CPU and 128Mi memory.
YAML Update:
```
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "250m"
```

## Step 6. Implement Probes
Add liveness and readiness probes for the nginx container:
Liveness probe: HTTP GET /healthz on port 80.
Readiness probe: TCP check on port 80.
YAML Update:
```
livenessProbe:
  httpGet:
    path: /healthz
    port: 80
  initialDelaySeconds: 3
  periodSeconds: 5
readinessProbe:
  tcpSocket:
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

## Step 7. Static Pod Implementation
### Deploy the same Pod as a static Pod by placing the YAML file in /etc/kubernetes/manifests/.
Steps:
- Copy the YAML file to the directory:
  - sudo cp ecommerce-pod.yaml /etc/kubernetes/manifests/
- Verify the kubelet automatically creates the Pod:
``` kubectl get pods```

## Step 8. Enable or control Networking
- Deploy a second Pod named backend running busybox.
- Verify communication between ecommerce-pod and backend using DNS.
Steps:

Create the backend Pod:
```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
```

Test communication:
```kubectl exec ecommerce-pod -- ping backend```


Now limit the communication between Pods to ensure that only authorized Pods can communicate with each other.

Create a Network Policy
Define a simple Network Policy that allows only Pods with a specific label to communicate with your Pod:
```
apiVersion: networking.k8s.io/v1  
kind: NetworkPolicy  
metadata:  
  name: allow-nginx  
spec:  
  podSelector:  
    matchLabels:  
      app: nginx  
  ingress:  
  - from:  
    - podSelector:  
        matchLabels:  
          app: nginx
```

Apply the Network Policy
Apply the policy using:

```kubectl apply -f network-policy.yaml  ```
Test Communication Between Pods
Deploy another Pod (e.g., curl-pod) in the same namespace and test if it can access the ecommerce Pod.

```kubectl run curl-pod --rm -it --image=curlimages/curl -- /bin/sh  
curl ecommerce-pod:80  
```

## Step 9. Test Pod Behavior
- Simulate a resource conflict by deploying the Pod on a node with limited resources.
- Deploy pod with the same name in same and different namespaces
- Test the liveness and readiness probes by making /healthz unavailable temporarily.


# References:
[Kubernetes Pods Documentation](https://kubernetes.io/docs/concepts/workloads/pods/)

# Essential kubectl Commands for Pods
- List Pods in the Current Namespace : ```kubectl get pods```
- List Pods Across All Namespaces : ``` kubectl get pods --all-namespaces```
- Describe Pod Details : ``` kubectl describe pod <pod-name>```
- View Logs of a Pod : ``` kubectl logs <pod-name>```
- For a specific container in a multi-container Pod: ``` kubectl logs <pod-name> -c <container-name>```
- Stream Logs in Real Time: ``` kubectl logs -f <pod-name>```
- Exec into a Running Pod : ``` kubectl exec -it <pod-name> -- /bin/bash```
- For a specific container in a multi-container Pod: ``` kubectl exec -it <pod-name> -c <container-name> -- /bin/bash ```
- List Pod Events : ``` kubectl get events --field-selector involvedObject.name=<pod-name> ```
- Delete a Pod : ``` kubectl delete pod <pod-name>```
- Get Pod YAML Configuration: ``` kubectl get pod <pod-name> -o yaml```
- Edit a Pod Configuration (use with caution; not for static Pods or deployments) : ``` kubectl edit pod <pod-name> ```
- List Pods in a Specific Namespace : ``` kubectl get pods -n <namespace> ```
- Describe Pod in a Specific Namespace : ``` kubectl describe pod <pod-name> -n <namespace> ```