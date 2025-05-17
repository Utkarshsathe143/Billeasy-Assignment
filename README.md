1. Start Minikube
minikube start --cpus=2 --memory=3500 --driver=docker --addons=ingress,metrics-server
2. Enable Ingress
minikube addons enable ingress
3. Create Namespaces
kubectl create namespace gateway
kubectl create namespace auth-service
kubectl create namespace data-service

4. Deploy Microservices
Used public Docker images to simulate microservices:
gateway: nginxdemos/hello
auth-service: kennethreitz/httpbin
data-service: hashicorp/http-echo

5. The using the Helm we created the files for gateway, auth service and data service.
helm create gateway
minikube addons enable ingress
helm install gateway ./gateway --namespace=gateway

we were getting an error as the image was not being pulled properly.
Warning  Failed     15m (x4 over 17m)     kubelet            Failed to pull image "nginxdemos/hello:1.16.0": Error response from daemon: manifest for nginxdemos/hello:1.16.0 not found: manifest unknown: manifest unknown
Warning  Failed     15m (x4 over 17m)     kubelet            Error: ErrImagePull
Warning  Failed     15m (x6 over 17m)     kubelet            Error: ImagePullBackOff
Normal   BackOff    2m13s (x64 over 17m)  kubelet            Back-off pulling image "nginxdemos/hello:1.16.0"

Then for troubleshooting, 
CLI snippet
PS C:\Users\sathe> kubectl get pods -n gateway
NAME                       READY   STATUS             RESTARTS   AGE
gateway-6d45576646-pwp4h   0/1     InvalidImageName   0          2m37s
PS C:\Users\sathe> kubectl describe pod gateway-6d45576646-pwp4h -n gateway

Then after the troubleshooting the image pulling was successful.
We changes the deployment file configuration to this code.
earlier there was some snippet **** added and that was causing the problem. 

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | ********* }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}

That snippet when present was causing the latest tag to come twice.
And after that when we removed that snippet and the latest tag it was coming like this.

  Warning  InspectFailed  12s (x4 over 37s)  kubelet            Failed to apply default image tag "nginxdemos/hello:latest:": couldn't parse image name "nginxdemos/hello:latest:": invalid reference format
  Warning  Failed         12s (x4 over 37s)  kubelet            Error: InvalidImageName
  
So we then removed the double quotes and it started working.

image:
  repository: nginxdemos/hello
  pullPolicy: IfNotPresent
  tag : latest
imagePullSecrets: []

PS C:\Users\sathe> helm upgrade gateway ./gateway --namespace=gateway
Release "gateway" has been upgraded. Happy Helming!
NAME: gateway
LAST DEPLOYED: Fri May 16 10:59:04 2025
NAMESPACE: gateway
STATUS: deployed
REVISION: 3
NOTES:
1. Get the application URL by running these commands:
  http://gateway.local/

CLI snippet

PS C:\Users\sathe> kubectl get pods -n gateway
NAME                       READY   STATUS    RESTARTS   AGE
gateway-55667ddf94-jtccs   1/1     Running   0          109s

6. minikube addons enable ingress
   Then we have to enable the ingress and the tunnel
7. minikube tunnel
   The gateway was now accessible.

helm create auth-service
helm install auth-service ./auth-service -n auth-service
helm upgrade auth-service ./auth-service -n auth-service - used when you have made some changes and you want them to take them into execution.
kubectl get hpa -n auth-service
CLI Snippet
NAME           REFERENCE                 TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
auth-service   Deployment/auth-service   <unknown>/80%   1         3         1          3m30s

As the metrics were not shown. for troubleshooting we used the commands 
kubectl describe hpa auth-service -n auth-service
kubectl describe pod auth-service-659bb68ff8-wzpfx -n auth-service

CLI snippet 

Events:
  Type     Reason         Age                  From     Message
  ----     ------         ----                 ----     -------
Warning  InspectFailed  22s (x348 over 78m)  kubelet  Failed to apply default image tag "nginx:": couldn't parse image name "nginx:": invalid reference format

Same error we were facing in the gateway. resolved it.
CLI snippet

CreationTimestamp:                                     Fri, 16 May 2025 16:25:58 +0530
Reference:                                             Deployment/auth-service
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  1% (1m) / 80%
Min replicas:                                          1
Max replicas:                                          3
Deployment pods:                                       1 current / 1 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)

TO check if the configuration is workinf properly we increased the traffic 
NAME           REFERENCE                 TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
auth-service   Deployment/auth-service   1%/80%    1         3         1          103m
PS C:\Users\sathe> kubectl exec -it -n auth-service auth-service-6889fdddd7-46djj -- /bin/sh

PS C:\Users\sathe> kubectl get hpa -n auth-service
NAME           REFERENCE                 TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
auth-service   Deployment/auth-service   133%/80%   1         3         2          112m
PS C:\Users\sathe> kubectl get hpa -n auth-service
NAME           REFERENCE                 TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
auth-service   Deployment/auth-service   201%/80%   1         3         3          113m
PS C:\Users\sathe> kubectl get hpa -n auth-service
NAME           REFERENCE                 TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
auth-service   Deployment/auth-service   201%/80%   1         3         3          113m
PS C:\Users\sathe> kubectl get hpa -n auth-service
NAME           REFERENCE                 TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
auth-service   Deployment/auth-service   201%/80%   1         3         3          114m
PS C:\Users\sathe> kubectl get hpa -n auth-service
NAME           REFERENCE                 TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
auth-service   Deployment/auth-service   35%/80%   1         3         3          114m

Then we created the data service.
helm create data-service
helm install data-service ./data-service -n data-service
PS C:\Users\sathe> kubectl get all -n data-service

CLI snippet

NAME                               READY   STATUS    RESTARTS   AGE
pod/data-service-6b789f8d9-vjmwg   1/1     Running   0          5m53s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/data-service   ClusterIP   10.108.69.61   <none>        5678/TCP   5m54s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/data-service   1/1     1            1           5m54s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/data-service-6b789f8d9   1         1         1       5m53s

Now for leaking Authorization headers

PS C:\Users\sathe> kubectl run curl --image=curlimages/curl -it --rm --restart=Never -- curl http://auth-service.auth-service.svc.cluster.local/headers -H "Authorization: Bearer FAKE-TOKEN"
{
  "headers": {
    "Accept": "*/*",
    "Authorization": "Bearer FAKE-TOKEN",
    "Host": "auth-service.auth-service.svc.cluster.local",
    "User-Agent": "curl/8.13.0"
  }
}
pod "curl" deleted
PS C:\Users\sathe> kubectl logs auth-service-5b94bfc8bb-dk4jw -n auth-service
[2025-05-17 04:30:36 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2025-05-17 04:30:36 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
[2025-05-17 04:30:36 +0000] [1] [INFO] Using worker: gevent
[2025-05-17 04:30:36 +0000] [9] [INFO] Booting worker with pid: 9

For monitoring we used grafanna and promehtheous 

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl port-forward svc/monitoring-grafana 3000:80

For IAM minio 

PS C:\Users\sathe> helm install minio minio/minio
Error: INSTALLATION FAILED: failed post-install: 1 error occurred:
        * timed out waiting for the condition

PS C:\Users\sathe> kubectl get pods -n minio
No resources found in minio namespace.
PS C:\Users\sathe> kubectl describe pod minio-0 -n minio
Error from server (NotFound): namespaces "minio" not found
PS C:\Users\sathe> helm install minio minio/minio
Error: INSTALLATION FAILED: cannot re-use a name that is still in use
PS C:\Users\sathe> kubectl get pods -n minio
No resources found in minio namespace.
PS C:\Users\sathe> kubectl get pods
NAME                   READY   STATUS    RESTARTS      AGE
minio-0                0/1     Pending   0             7m55s
minio-1                0/1     Pending   0             7m55s
minio-10               0/1     Pending   0             7m54s
minio-11               0/1     Pending   0             7m54s
minio-12               0/1     Pending   0             7m55s
minio-13               0/1     Pending   0             7m54s
minio-14               0/1     Pending   0             7m54s
minio-15               0/1     Pending   0             7m54s
minio-2                0/1     Pending   0             7m55s
minio-3                0/1     Pending   0             7m55s
minio-4                0/1     Pending   0             7m55s
minio-5                0/1     Pending   0             7m55s
minio-6                0/1     Pending   0             7m55s
minio-7                0/1     Pending   0             7m55s
minio-8                0/1     Pending   0             7m54s
minio-9                0/1     Pending   0             7m55s
minio-post-job-6v8bh   1/1     Running   5 (90s ago)   7m55s

For troubleshooting we used.
kubectl describe pod minio-0
Warning  FailedScheduling  8m6s   default-scheduler  0/1 nodes are available: 1 Insufficient memory. preemption: 0/1 nodes are available: 1 No preemptionvictims found for incoming pod..
Warning  FailedScheduling  2m59s  default-scheduler  0/1 nodes are available: 1 Insufficient memory. preemption: 0/1 nodes are available: 1 No preemptionvictims found for incoming pod..
PS C:\Users\sathe> helm uninstall minio -n default


I have added all the Screenshots and video recordings and the json document for monitoring.
