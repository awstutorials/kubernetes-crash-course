## Installing Microservices application in AWS EKS

**Login to AWS CLI**

```export AWS_DEFAULT_PROFILE=<profile_name>```

**Prerequisites**
0. Install aws cli latest version
 - [Install/Upgrade cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-mac.html#cliv2-mac-install-cmd)
1. Install eksctl
 - Install the Weaveworks Homebrew tap.
    ```
    brew tap weaveworks/tap
    ```
2. Install or upgrade eksctl
    ```
    brew install weaveworks/tap/eksctl
    ```
- For Upgrade
```
brew upgrade eksctl && brew link --overwrite eksctl
```
3. Test that your installation was successful with the following command.
```
eksctl version
```
- Output : The GitTag version should be at least 0.10.2

**How to create Cluster**
```
eksctl create cluster \
--name prod \
--version 1.14 \
--region eu-west-1 \
--nodegroup-name standard-prd-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 7 \
--managed \
--asg-access
```

```eksctl create cluster --name test-auto --version 1.14 --without-nodegroup```
- Verify EKS Clusters. Cluster provisioning usually takes between 10 and 15 minutes. When your cluster is ready, test that your kubectl configuration is correct.
```
kubectl get svc
```
    If you receive the error "aws-iam-authenticator": executable file not found in $PATH, your kubectl isn't configured for Amazon EKS. For more information, see [Installing aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html).

    If you receive any other authorization or resource type errors, see [Unauthorized or Access Denied (kubectl)](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html#unauthorized) in the troubleshooting section.

* Use the AWS CLI update-kubeconfig command to create or update your kubeconfig for your cluster.

```
aws eks --region region update-kubeconfig --name cluster_name
```
**Metrics Server Installation Procedure**
```
DOWNLOAD_URL=$(curl --silent "https://api.github.com/repos/kubernetes-sigs/metrics-server/releases/latest" | jq -r .tarball_url)
DOWNLOAD_VERSION=$(grep -o '[^/v]*$' <<< $DOWNLOAD_URL)
curl -Ls $DOWNLOAD_URL -o metrics-server-$DOWNLOAD_VERSION.tar.gz
mkdir metrics-server-$DOWNLOAD_VERSION
tar -xzf metrics-server-$DOWNLOAD_VERSION.tar.gz --directory metrics-server-$DOWNLOAD_VERSION --strip-components 1
kubectl apply -f metrics-server-$DOWNLOAD_VERSION/deploy/1.8+/
kubectl get deployment metrics-server -n kube-system
```

Viewing Raw Metrics
```
kubectl get --raw /metrics
```

**Install Promethus**

a. Create a Prometheus namespace.
```
kubectl create namespace prometheus
```
b. Deploy Prometheus.
```
helm install prometheus stable/prometheus \
   --namespace prometheus \
   --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```
c. Verify that all of the pods in the prometheus namespace are in the READY state.
```
kubectl get pods -n prometheus
```
d. Use kubectl to port forward the Prometheus console to your local machine.
```
kubectl --namespace=prometheus port-forward deploy/prometheus-server 9090
```
e. Verify : Choose a metric from the - insert metric at cursor menu, then choose Execute. Choose the Graph tab to show the metric over time. The following image shows container_memory_usage_bytes over time.

**Kubernetes Dashboard**
1. Metrics Server is also input for Dashboard which is already installed. (refer above)
2. Deploy Dashboard
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
```
3. Create an eks-admin Service Account and Cluster Role Binding
    - Create a file called eks-admin-service-account.yaml with the text below. This manifest defines a service account and cluster role binding called eks-admin.
   ```
   apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: eks-admin
        namespace: kube-system
      ---
      apiVersion: rbac.authorization.k8s.io/v1beta1
      kind: ClusterRoleBinding
      metadata:
        name: eks-admin
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-admin
      subjects:
      - kind: ServiceAccount
        name: eks-admin
        namespace: kube-system
   ``` 
4. Apply the service account and cluster role binding to your cluster.
```
kubectl apply -f eks-admin-service-account.yaml
```
5. Connect to the Dashboard
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```
6. Copy token
7. Start the kubectl proxy
```
kubectl proxy
```
8. Access the dashboard endpoint, open the following link with a web browser: [http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login)

**Install Microservices/Applications**
1. Install Hello-World-rest-api

```
cd 01-hello-world-rest-api

kubectl create deployment hello-world-rest-api --image=in28min/hello-world-rest-api:0.0.1.RELEASE
kubectl expose deployment hello-world-rest-api --type=LoadBalancer --port=8080
```
- Verify Installation
```
kubectl get svc
```
Copy the Load Balancer url, Usually Load Balance initialization will take sometime so wait for 1 minutes.
 - http://${loadbalancerurl}:8080/hello-world

2. Install todo-web-application

```
cd 02-todo-web-application-h2

kubectl create deployment todowebapp-h2 --image=in28min/todo-web-application-h2:0.0.1-SNAPSHOT
kubectl expose deployment todowebapp-h2 --type=LoadBalancer --port=8080
```

- Verify Installation
```
kubectl get svc
```
Copy the Load Balancer url, Usually Load Balance initialization will take sometime so wait for 1 minutes.
 - http://${loadbalancerurl}:8080/

3. Install todo-web-application-mysql  
- Please check file : mysql-database-persistentvolumeclaim-aws.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cloud-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```
- Here We configured AWS - EBS volume for mysql data storage - stop the application and restart mysql would result in persistent volume.
 
```
cd kubernetes-crash-course/03-todo-web-application-mysql/backup/02-final-backup-at-end-of-course

kubectl apply -f mysql-database-persistentvolumeclaim-aws.yaml,mysql-deployment.yaml,mysql-service.yaml
kubectl apply -f configmap.yaml
kubectl create secret generic todo-web-application-secrets-1 --from-literal=RDS_PASSWORD=dummytodos
kubectl apply -f todo-web-application-deployment.yaml,todo-web-application-service.yaml

```
- Verify Installation
```
kubectl get svc
```
- Copy the Load Balancer url, Usually Load Balance initialization will take sometime so wait for 1 minutes.
  - http://${loadbalancerurl}:8080/

3a. Install Kubernetes Dashboard to show little metrics

3b. Delete all the applications and Database installed so far.
```
kubectl get all
kubectl delete all -l app=<app_name>
```


4a. **[Ingress installation](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)**

   Create an IAM policy called ALBIngressControllerIAMPolicy for your worker node instance profile that allows the ALB Ingress Controller to make calls to AWS APIs on your behalf. Use the following AWS CLI commands to create the IAM policy in your AWS account
   a. Download the policy document from GitHub.
   ```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.3/docs/examples/iam-policy.json
   ```

   b. Create the policy.
   ```
    aws iam create-policy \
      --policy-name ALBIngressControllerIAMPolicy \
      --policy-document file://iam-policy.json
   ```
  c. Get the IAM role name for your worker nodes. Use the following command to print the aws-auth configmap.
  
  ```kubectl -n kube-system describe configmap aws-auth```
     
  d. Attach the new ALBIngressControllerIAMPolicy IAM policy to each of the worker node IAM roles you identified earlier with the following command.
  
  ```
    aws iam attach-role-policy \
        --policy-arn arn:aws:iam::182388080935:policy/ALBIngressControllerIAMPolicy \
        --role-name eksctl-test-nodegroup-standard-wo-NodeInstanceRole-UHWEGLWODWUJ
        --cluster-name=test
        --aws-vpc-id=vpc-0939c26c15793e7a3
  ```
   e. Create a service account, cluster role, and cluster role binding for the ALB Ingress Controller to use with the following command.
    
  ```
  kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.3/docs/examples/rbac-role.yaml
  ```
   f. Deploy the ALB Ingress Controller with the following command.
   ```
   kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.3/docs/examples/alb-ingress-controller.yaml
   ```
   g. Open the ALB Ingress Controller deployment manifest for editing with the following command.
   ```
   kubectl edit deployment.apps/alb-ingress-controller -n kube-system
   ```
    
   h. Add the cluster name, VPC ID, and AWS Region name for your cluster after the --ingress-class=alb line and then save and close the file.
   ```
        spec:
              containers:
              - args:
                - --ingress-class=alb
                - --cluster-name=my_cluster
                - --aws-vpc-id=vpc-03468a8157edca5bd
                - --aws-region=us-west-2
   ```  
   i. Verify Ingress
   
   ```kubectl get ingress/2048-ingress -n 2048-game```
   
    
   j. Delete the Ingress
   
   ```kubectl delete ingress/2048-ingress -n 2048-game``` 
    
4. Install 04-currency-exchange-microservice-basic

```
cd 04-currency-exchange-microservice-basic
kubectl apply -f deployment.yaml

```
5. Install 05-currency-conversion-microservice-basic

```
cd 05-currency-conversion-microservice-basic

kubectl apply -f deployment.yaml
install ingress controller and get the ip address
kubectl apply -f ingress_aws.yaml
kubectl get ingress/ingress-aws
http://loadbalancerURL/currency-conversion/from/EUR/to/INR/quantity
ingress controller installation

```
6. 06-currency-conversion-microservice-cloud

```
cd 06-currency-conversion-microservice-cloud
kubectl apply -f deployment.yaml
kubectl apply -f 02-rbac.yaml
```

## Application and Cluster Logging

Logging is implemented via EFK Elastic Search - FluentD - Kibana

- Installation Steps
- Video Link : [Chapter 4-k8s](https://www.youtube.com/watch?v=K3OgTEUJQ14) - Time from 30:00 (10 mins)
- Navigate to directory fluentd-elasticsearch and run
```
kub ectl apply -f .
```

** Get Kibana URL **
```
kubectl get svc -n kube-system
```

You should see kiabana-logging - Copy Load balancer url along with port : 5601

## Streamming Logs - CloudWatch

```
aws eks --region region-code update-cluster-config --name prod \
--logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```
Note: When you use Amazon EKS control plane logging, you're charged standard Amazon EKS pricing for each cluster that you run. You are charged the standard CloudWatch Logs data ingestion and storage costs for any logs sent to CloudWatch Logs from your clusters. You are also charged for any AWS resources, such as Amazon EC2 instances or Amazon EBS volumes, that you provision as part of your cluster.


**Install Istio**
```
kubectl create namespace istio-system
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.2 sh -
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
helm template install/kubernetes/helm/istio --name istio --set global.mtls.enabled=false --set tracing.enabled=true --set kiali.enabled=true --set grafana.enabled=true --namespace istio-system > istio.yaml
kubectl apply -f istio.yaml
kubectl label namespace default istio-injection=enabled
kubectl create secret generic kiali -n istio-system --from-literal=username=admin --from-literal=passphrase=admin

kubectl get secret -n istio-system kiali
kubectl create secret generic kiali -n istio-system --from-literal=username=admin --from-literal=passphrase=admin
kubectl port-forward \
    $(kubectl get pod -n istio-system -l app=kiali \
    -o jsonpath='{.items[0].metadata.name}') \
    -n istio-system 20001
http://localhost:20001
kubectl -n istio-system port-forward \
    $(kubectl -n istio-system get pod -l app=grafana \
    -o jsonpath='{.items[0].metadata.name}') 3000
http://localhost:3000
kubectl port-forward -n istio-system \
    $(kubectl get pod -n istio-system -l app=jaeger \
    -o jsonpath='{.items[0].metadata.name}') 16686
http://localhost:16686
```
Verify URLS:
```
http://<lburl>/hello-world
http://<lburl>/currency-conversion/from/EUR/to/INR/quantity/29987
http://<lburl>/currency-exchange/from/EUR/to/INR
```

Verify Jagger by creating some load
```watch -n 0.1 curl http://a975654c512ae11ea998202d1279c960-877389647.eu-west-1.elb.amazonaws.com//currency-conversion/from/EUR/to/INR/quantity/29987```


**Create Cluster Autoscaler**

1. Deploy the Cluster Autoscaler to your cluster with the following command.
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```
2. Add the cluster-autoscaler.kubernetes.io/safe-to-evict annotation to the deployment with the following command.
```
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```
3.Edit the Cluster Autoscaler deployment with the following command.
```
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```
    Edit the cluster-autoscaler container command to replace <YOUR CLUSTER NAME> with your cluster's name, and add the following options.
    * --balance-similar-node-groups
    * --skip-nodes-with-system-pods=false
```
    spec:
      containers:
      - command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
```
4. Set the Cluster Autoscaler image tag to the version. Open the Cluster Autoscaler [releases](https://github.com/kubernetes/autoscaler/releases) page in a web browser and find the Cluster Autoscaler version that matches your cluster's Kubernetes major and minor version
```
kubectl -n kube-system set image deployment.apps/cluster-autoscaler cluster-autoscaler=k8s.gcr.io/cluster-autoscaler:v1.14.6
```
5. After you have deployed the Cluster Autoscaler, you can view the logs and verify that it is monitoring your cluster load.
   View your Cluster Autoscaler logs with the following command.
   ```
   kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
   ```

**Delete Resources belong to namespace**

```kubectl delete all --all -n {namespace}```

**Delete Cluster**

1. First delete any service which has external ip address/Load Balance

```
kubectl get svc --all-namespaces
kubectl delete svc service-name
```

2. Delete the cluster and its associated worker nodes with the following command, replacing prod with your cluster name.

```
eksctl delete cluster --name prod
```
       
rolearn: arn:aws:iam::182388080935:role/eksctl-test-nodegroup-standard-wo-NodeInstanceRole-1WSUER18EFW02

eksctl-test-nodegroup-standard-wo-NodeInstanceRole-UHWEGLWODWUJ
