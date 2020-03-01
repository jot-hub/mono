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