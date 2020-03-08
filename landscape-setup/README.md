Source: https://cloud.google.com/solutions/jenkins-on-kubernetes-engine-tutorial/

Note: Tiller not necessary from Helm 3.0, so some steps are ignored

1. Create a Kubernetes cluster.
> gcloud container clusters create mono-cluster-1   --machine-type n1-standard-2 --num-nodes 2 --cluster-version 1.15.8-gke.3

2. Make current user a cluster-admin
> kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin \
        --user=$(gcloud config get-value account)

3. Setup helm repo
> helm repo add stable https://kubernetes-charts.storage.googleapis.com

4. Install Jenkins with values from jenkins/values.yaml from https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
> helm install -n ci helmed-jenkins stable/jenkins --values https://raw.githubusercontent.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes/master/jenkins/values.yaml --wait

5. Get secret from cluster for jenkins admin password
> printf $(kubectl get secret --namespace ci helmed-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo

6. Port-forward to Jenkins master to access Jenkins
> export POD_NAME=$(kubectl get pods --namespace ci -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=helmed-jenkins" -o jsonpath="{.items[0].metadata.name}")

> kubectl --namespace ci port-forward $POD_NAME 8080:8080

7. GCE ingress does not support target url rewrite - i.e. when an ingress is setup based on `/path`, the backend service gets the request as `/path` and not just `/`. Example: If `/jenkins` is the path to a target service, the target service gets the path as as `/jenkins` and not just `/`. This is bad. https://github.com/kubernetes/ingress-gce/issues/109
For that reason, disable `GCE Ingress` and enable `nginx Ingress`.
> gcloud container clusters update mono-cluster-1 --update-addons HttpLoadBalancing=DISABLED

8. Install nginx ingress controller
> helm install --name nginx-ingress stable/nginx-ingress --set rbac.create=true --set controller.publishService.enabled=true

9. If you want Jenkins to be accessible externally, so that github hooks can reach and trigger Jenkins jobs, then Jenkins instance has to be exposed - exposing it via an ingress is better (instead of exposing it with a load balancer especially for Jenkins instance. The idea is to use the same ingress for exposing additional services in the future).
> kubectl apply -f ingress.yaml