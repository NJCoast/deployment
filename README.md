# Documentation on NJcoast kubernetes deployment system

## Table of Contents
* [Background](#background)
* [Monitoring](#monitoring)
* [Command Line Tool Installation](#command-line)
* [Kubernetes Production Deployment](#production)
* [Kubernetes CRC Staging Deployment](#staging)
* [Executing Django Database Migrations](#django-migrations)

## <a name="background"></a> Background
NJcoast is designed to be deployed to the [Amazon AWS](https://aws.amazon.com/) AWS cloud computing environment. The deployment mechanism utilizes the command-line interface and requires background knowledge of the local OS (Windows, MacOS, Linux) to install the appropriate command line tools. This deployment workflow has been tested under MacOS terminal, BASH and ZSH shells, other command line environments may differ. NJcoast uses the [Kubernetes](https://kubernetes.io/) container orchestration tool to deploy all of the NJCoast micro-services. In addition, NJcoast uses the [Helm](https://helm.sh/) package manager for Kubernetes that enables the definition, installation, and upgrade even the most complex Kubernetes applications. You will also need [aws configuration and registry information](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) for the account to which the infrastructure is being deployed.  Three different command line workflows are contained in these instructions. One for deployment to production in AWS. One for deploying to a local Kubernetes development instance and lastly one for deploying to a staging and testing environment. In addition, commands for doing in-place upgrades of the infrastructure once deployed. 

**Note: Execution of the first-time deployment instructions on an existing instance may result in loss of data and the infrastructure being overwritten. The Upgrade instructions are to be utilized on an existing instance that has been deployed to AWS.**

*James: This may need to be flushed out with which repositories pull from where...*
The Helm orchestration system automatically builds and installs software from the [NJcoast](https://github.com/njcoast) organization. Branches in the software repositories are used to determine which which software version is installed in which deployment. The NJcoast and Geonode [cyberspatial-ui](https://github.com/NJCoast/cyberspatial-ui) git development branch will be deployed to the development deployment described below. Some tools installed from NJcoast organization Github repositories only contain a master branch, in those cases the deployment will pull from the master branch.

### <a name="monitoring"></a> Monitoring
NJcoast deploys [Grafana](https://grafana.com/), an open source analytics, monitoring and data visualization platform, to provide a user interface for monitoring of NJcoast container instances. Grafana is community driven, and extensible providing the ability to add additional monitoring dashboards as needed. Two data sources for Grafana are deployed as container through Kubernetes. [Prometheus](https://prometheus.io/docs/introduction/overview/), is a pull-based system and service monitoring system that collects metrics from targets that are pre-defined and has support for Kubernetes deployments. Second, a data source for [AWS Cloudwatch](https://aws.amazon.com/cloudwatch/). Detailed Instructions for adding AWS Cloudwatch as [a data source for Grafana](https://grafana.com/docs/features/datasources/cloudwatch/) require configuration of the permissions for EC2 tags/instances/regions and require configuration through a policy specification and a AWS credentials file to access account statistics. More details of configuring IAM roles can be found in [Amazon's AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html).

*James: We probably need to add some instructions on how to access the Grafana dashboard, is this done through a proxy?*

## <a name="command-line"></a> Command Line Tool Installation
First, kubectl and helm command line tools must be installed on the local operating system that is managing the deployment. Detailed installation instructions for various platforms including windoes is included in the [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [helm](https://helm.sh/docs/using_helm/) documentation. For MacOS, the [homebrew](https://brew.sh/) package management system is the preferred installation mechanism. 

From [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-macos)
```bash
brew install kubernetes-cli
```

From [helm](https://helm.sh/docs/using_helm/#installing-helm)
```bash
brew install kubernetes-helm
```

*JAMES: Not sure about this bit. How does the kubectl utilize the aws tools?*
In addition, the [aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html) is needed to provide authentication to your Kubernetes cluster through the AWS IAM Authenticator for Kubernetes. You can configure the stock kubectl client to work with Amazon EKS by installing the AWS IAM Authenticator for Kubernetes and modifying your kubectl configuration file to use it for authentication.

```bash
brew install aws-iam-authenticator
```


## <a name="production"></a> Kubernetes Production Deployment
The following command line workflow deploys the NJcoast system to a **PRODUCTION** environment in AWS. These are the "first time" deployment instructions.

### Dashboard
These commands deploy the [Kubernetes Dashboard](https://github.com/kubernetes/dashboard) general purpose web based UI for Kubernetes Clusters. It allows administrators to manage applications running in the cluster and troubleshoot them, as well as manage the cluster itself.

```
helm install --namespace kube-system --name heapster stable/heapster --values heapster.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

#### User
These commands create a [Kubernetes dashboard user](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user) to provide secure access to the dashboard by default.

```
kubectl apply -f dashboard/user.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user-token | cut -f1 -d' ') | grep "token:" | cut -d' ' -f7
```

#### Access
These commands make the [Kubernetes Dashboard remotely available](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) through the kubectl command line tool. The command makes the Kubernetes Dashboard available through the local machines webhost by pointing a web browser url at the http://127.0.0.1:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/ address.

```
sudo kubectl proxy
```

Go to http://127.0.0.1:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/. Use token login and paste token from user setup command.

### Helm
These commands deploy the [helm charts](https://helm.sh/docs/developing_charts/) responsible for deploying all of the NJcoast infrastructure.

```
kubectl apply -f helm-rbac.yaml
helm init --service-account tiller
```

### nginx
These commands configure the [nginx](https://www.nginx.com/) web server container that provides a proxy/load balancing for all NJcoast public service endpoints. 

```
helm install --namespace kube-system --name ingress stable/nginx-ingress --set controller.hostNetwork=true,controller.tolerations[0].effect=NoSchedule,controller.tolerations[0].key=node-role.kubernetes.io/master,controller.nodeSelector."kubernetes\.io/hostname"="njcoast1\.virtual\.crc\.nd\.edu"
```

### Certificate Manager
These commands deploy the Kubernetes SSL [Certificate Manager](https://docs.cert-manager.io/en/latest/#) to manage NJcoast certificate generation through the [AWS Certificate Manager](https://aws.amazon.com/certificate-manager/).

```
helm install --name tls --namespace kube-system stable/cert-manager
kubectl apply -f certificate-issuer.yaml
```

### AWS Docker Registry Credentials
```
kubectl apply -f aws-registry/secret.yaml
kubectl apply -f aws-registry/replication.yaml
```

### Grafana
These commands deploy [Grafana](https://grafana.com/) as described in the [Monitoring section](#monitoring).

```
kubectl apply -f grafana/certificate.yaml
kubectl apply -f grafana/persistant-volumes.yaml
helm install --name grafana stable/grafana -f grafana/values.yaml
```

### Prometheus Operator
These commands deploy [Prometheus](https://coreos.com/operators/prometheus/docs/latest/user-guides/getting-started.html) as described in the [Monitoring section](#monitoring).

```
helm repo add coreos https://s3-eu-west-1.amazonaws.com/coreos-charts/stable/ 
helm install coreos/prometheus-operator --name prometheus-operator --namespace monitoring -f operator.yaml 
helm install coreos/kube-prometheus --name kube-prometheus --namespace monitoring -f prometheus.yaml
```

### Cloudwatch Logging
These commands deploy the [AWS Cloudwatch utilities](https://github.com/helm/charts/tree/master/incubator/fluentd-cloudwatch) as described in the [Monitoring section](#monitoring).

```
helm install --name logging fluentd-cloudwatch/
```

### <a name="staging"></a> Staging

#### Install
```
kubectl create namespace staging
kubectl apply -f staging/certificate.yaml
kubectl apply -f staging/persistant-volumes.yaml
kubectl apply -f staging/monitoring.yaml
helm install --name staging --namespace staging cyberspatial --values staging/configuration.yaml
```

#### Migrations
These commands are necessary for all deployments to populate the database with default tables and values. These execute the [django migrations uttility](https://docs.djangoproject.com/en/1.7/topics/migrations/) in the cyberspatial-django container.
```
kubectl -n staging exec -c cyberspatial-django $(kubectl -n staging get pod | grep django | cut -d' ' -f1) -- /app/manage.py migrate --noinput
kubectl -n staging exec -c cyberspatial-django $(kubectl -n staging get pod | grep django | cut -d' ' -f1) -- /app/manage.py loaddata sample_admin
kubectl -n staging exec -c cyberspatial-django $(kubectl -n staging get pod | grep django | cut -d' ' -f1) -- /app/manage.py loaddata default_oauth_apps_docker
kubectl -n staging exec -c cyberspatial-django $(kubectl -n staging get pod | grep django | cut -d' ' -f1) -- /app/manage.py loaddata initial_data
```

## Upgrade
The following command line workflow upgrades an **EXISTING** NJcoast deployment for each of the three deployment types enumerated previously.



### Staging
These commands upgrade the CRC staging instance.
```
helm upgrade staging cyberspatial --values staging/configuration.yaml
```