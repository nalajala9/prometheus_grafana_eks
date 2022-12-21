# Prometheus and Grafana Installation on EKS

## Step-1: Create IAM policy for EBS:

* Go to Services -> IAM
* Create a Policy
* Select JSON tab and copy paste the below JSON

```


{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteTags",
        "ec2:DeleteVolume",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}

```


* Click on Review Policy and provide name for the policy (eg:Amazon_EBS_CSI_Driver)
* Click on Create Policy


      
 


## Step-2: Get the IAM role of WorkerNode and attach the above policy to the role

    kubectl -n kube-system describe configmap aws-auth

>  Note the role arn

* Go to Services -> IAM -> Roles
* Search for the role arn that we got earlier
* Click on Permissions tab
* Click on Attach Policies
* Search for created plociy earlier(Amazon_EBS_CSI_Driver) and click on Attach Policy

## Step-3: Deploy Amazon EBS CSI Driver

### To deploy the CSI driver:


    kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.13"

### Verify driver is running:

    kubectl get pods -n kube-system
    
## Step-4: Install helm cli

    curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

> Verify the version

    helm version --short

## Step-5:  Adding prometheus and grafana repos

### Add prometheus Helm repo
    
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

### Add grafana Helm repo
    
    helm repo add grafana https://grafana.github.io/helm-charts

 > Verify all repos added in your local 
 
    helm repo list
          
## Step-6: Prometheus Installation


```

kubectl create namespace 
        
helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"


```

> Make note of the prometheus endpoint in helm response (you will need this later). It should look similar to below: prometheus-server.prometheus.svc.cluster.local

> Check if Prometheus components deployed as expected

      kubectl get all -n prometheus
      
> In order to access the Prometheus server URL, we are going to use the kubectl port-forward command to access the application.

      kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090( not able to access in private subnets , need to use type of NodePort and LoadBalacer)
      

## Step-6: Grafana Installation

> Create YAML file called grafana.yaml with following commands:
          
          
```
mkdir ${HOME}/environment/grafana

cat << EoF > ${HOME}/environment/grafana/grafana.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true
EoF

```

> Create namespace

    kubectl create namespace grafana

```
helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='EKS!sAWSome' \
    --values ${HOME}/environment/grafana/grafana.yaml \
    --set service.type=LoadBalancer

```

> Run the following command to check if Grafana is deployed properly:

      kubectl get all -n grafana
      
> You can get Grafana ELB URL using this command. Copy & Paste the value into browser to access Grafana web UI.


      export ELB=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

      echo "http://$ELB"
      
      or check in  kubectl get all -n grafana and copy load balancer dns 
      
 > Note: It can take several minutes before the ELB is up, DNS is propagated and the nodes are registered.


> When logging in, use the username admin and get the password hash by running the following:

      kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

## Step-7 : DASHBOARDS in Grafana


> Log in to Grafana dashboard using credentials supplied during configuration and We will import community created dashboard for this tutorial.

> For creating a dashboard to monitor the cluster: 

```

Click '+' button on left panel and select ‘Import’.
Enter 3119 dashboard id under Grafana.com Dashboard.
Click ‘Load’.
Select ‘Prometheus’ as the endpoint under prometheus data sources drop down.
Click ‘Import’.

This will show monitoring dashboard for all cluster nodes

```

> Pods Monitoring Dashboard

```
Click '+' button on left panel and select ‘Import’.
Enter 6336 dashboard id under Grafana.com Dashboard.
Click ‘Load’.
Enter Kubernetes Pods Monitoring as the Dashboard name.
Click change to set the Unique identifier (uid).
Select ‘Prometheus’ as the endpoint under prometheus data sources drop down.s
Click ‘Import’.

```


## Step-7: Cleaning up 

```
helm uninstall prometheus --namespace prometheus
kubectl delete ns prometheus

helm uninstall grafana --namespace grafana
kubectl delete ns grafana

rm -rf ${HOME}/environment/grafana

```








