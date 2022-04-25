## Pre-Reqs
1. [kind](https://kind.sigs.k8s.io)
2. AWS credentials
3. [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)
4. [jq](https://stedolan.github.io/jq/download/)

**PLEASE NOTE:** For macOS users when using `kind` you may need to [increase the memory available](https://docs.docker.com/docker-for-mac/#resources) for containers (recommend 6Gb). `kind` is not designed for production use and is used for the creation of a temporary bootstrap cluster used to provision a target management cluster on the selected infrastructure provider, in this case AWS.

### Install clusterctl (non-mac users [check here](https://cluster-api-aws.sigs.k8s.io/getting-started.html#install-clusterctl))
```
brew install clusterctl
clusterctl version
```

### Create CAPI Management Cluster
```
kind create cluster
```

### Set AWS Credentials
```
export AWS_REGION=
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_ACCESS_KEY=
AWS_SESSION_TOKEN=                  # Optional - If you are using Multi-Factor Auth.
```

## Install clusterawsadm and Bootstrap AWS IAM resources for CAPI
Get the latest [clusterawsadm](https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases) and place it in your path.
The `clusterawsadm` command line utility assists with identity and access management (IAM) for Cluster API Provider AWS. It takes the credentials that you set as environment variables and uses them to create a CloudFormation stack in your AWS account with the correct IAM resources.
```
clusterawsadm bootstrap iam create-cloudformation-stack
```
The output will be similar to:
```
Attempting to create AWS CloudFormation stack cluster-api-provider-aws-sigs-k8s-io

Following resources are in the stack: 

Resource                  |Type                                                                                  |Status
AWS::IAM::InstanceProfile |control-plane.cluster-api-provider-aws.sigs.k8s.io                                    |CREATE_COMPLETE
AWS::IAM::InstanceProfile |controllers.cluster-api-provider-aws.sigs.k8s.io                                      |CREATE_COMPLETE
AWS::IAM::InstanceProfile |nodes.cluster-api-provider-aws.sigs.k8s.io                                            |CREATE_COMPLETE
AWS::IAM::ManagedPolicy   |arn:aws:iam::031814252212:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io   |CREATE_COMPLETE
AWS::IAM::ManagedPolicy   |arn:aws:iam::031814252212:policy/nodes.cluster-api-provider-aws.sigs.k8s.io           |CREATE_COMPLETE
AWS::IAM::ManagedPolicy   |arn:aws:iam::031814252212:policy/controllers.cluster-api-provider-aws.sigs.k8s.io     |CREATE_COMPLETE
AWS::IAM::ManagedPolicy   |arn:aws:iam::031814252212:policy/controllers-eks.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::Role            |control-plane.cluster-api-provider-aws.sigs.k8s.io                                    |CREATE_COMPLETE
AWS::IAM::Role            |controllers.cluster-api-provider-aws.sigs.k8s.io                                      |CREATE_COMPLETE
AWS::IAM::Role            |eks-controlplane.cluster-api-provider-aws.sigs.k8s.io                                 |CREATE_COMPLETE
AWS::IAM::Role            |nodes.cluster-api-provider-aws.sigs.k8s.io                                            |CREATE_COMPLETE
```

Create the base64 encoded credentials using `clusterawsadm`. This command uses your environment variables and encodes them in a value to be stored in a k8s secret.
```
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)
```

### Initialise CAPI Management Cluster with AWS provider (CAPA)
The `clusterctl` command accepts as input a [list of providers to install](https://cluster-api.sigs.k8s.io/user/quick-start.html#initialization-for-common-providers). When executed for the first time, `clusterctl init` automatically adds to the list the cluster-api core provider, and if unspecified, it also adds the `kubeadm` bootstrap and `kubeadm` control-plane providers.
```
clusterctl init --infrastructure aws
```
The output should be similar to:
```
Fetching providers
Installing cert-manager Version="v1.5.3"
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v1.1.3" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v1.1.3" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v1.1.3" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-aws" Version="v1.4.1" TargetNamespace="capa-system"

Your management cluster has been initialized successfully!
```

### Configure AWS CAPI Provider (CAPA)
Depending on the infrastructure provider, there are [additional prerequisites](https://cluster-api.sigs.k8s.io/user/quick-start.html#required-configuration-for-common-providers) that need to be satisfied before configuring a cluster with CAPI.
You can look at the `clusterctl generate cluster` command documentation to explore the list of variables required by the different cluster templates. For AWS the following additional pre-reqs are needed - you can check the [AWS provider prerequisites](https://cluster-api-aws.sigs.k8s.io/topics/using-clusterawsadm-to-fulfill-prerequisites.html) for more details.

#### Create SSH key pair
```
aws ssm put-parameter --name "/sigs.k8s.io/cluster-api-provider-aws/ssh-key" \
  --type SecureString \
  --value "$(aws ec2 create-key-pair --key-name default --output json | jq .KeyMaterial -r)"
```

#### Set AWS Workload Cluster Parameters
```
export AWS_REGION=eu-west-2
export AWS_SSH_KEY_NAME=default

export AWS_CONTROL_PLANE_MACHINE_TYPE=t3a.large
export AWS_NODE_MACHINE_TYPE=t3a.large
```

### Generating the Cluster Configuration
```
cd dev/
clusterctl generate cluster capi-aws-quickstart \
  --kubernetes-version v1.23.3 \
  --control-plane-machine-count=3 \
  --worker-machine-count=3 \
  > capi-aws-quickstart.yaml
```
This creates a YAML file named `capi-aws-quickstart.yaml` with a predefined list of Cluster API objects: Cluster, Machines, Machine Deployments, etc. The file can be eventually modified using an editor, just like you would modify any application yaml file.

### Create the workload cluster
```
kubectl apply -f capi-aws-quickstart.yaml
```
The output should be similar to:
```
cluster.cluster.x-k8s.io/capi-aws-quickstart created
awscluster.infrastructure.cluster.x-k8s.io/capi-aws-quickstart created
kubeadmcontrolplane.controlplane.cluster.x-k8s.io/capi-aws-quickstart-control-plane created
awsmachinetemplate.infrastructure.cluster.x-k8s.io/capi-aws-quickstart-control-plane created
machinedeployment.cluster.x-k8s.io/capi-aws-quickstart-md-0 created
awsmachinetemplate.infrastructure.cluster.x-k8s.io/capi-aws-quickstart-md-0 created
kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io/capi-aws-quickstart-md-0 created
```
The cluster will now start provisioning. You can check status:
```
kubectl get cluster
```
You can also get an “at glance” view of the cluster and its resources by running:
```
clusterctl describe cluster capi-aws-quickstart
```
To verify the first control plane is up:
```
kubectl get kubeadmcontrolplane
```
**PLEASE NOTE:** The control plane won’t be `Ready` until we install a CNI.

#### Deploy the CNI to the cluster

Make sure that the first control plane node is up and running
```
watch kubectl get kubeadmcontrolplane
```
Once there is at least 1 Replica, we can get the `kubeconfig` file:
```
NAME                                CLUSTER               INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE     VERSION
capi-aws-quickstart-control-plane   capi-aws-quickstart                                        1                  1         1             4m52s   v1.23.3
```
```
clusterctl get kubeconfig capi-aws-quickstart > capi-aws-quickstart.kubeconfig
```
Then we can install the CNI into the new workload cluster:
```
kubectl --kubeconfig=./capi-aws-quickstart.kubeconfig \
  apply -f https://docs.projectcalico.org/v3.21/manifests/calico.yaml
```
The output should be similar to:
```
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```
Once the CNI is deployed you can watch the control plane and worker nodes becoming available and ready.
```
watch kubectl --kubeconfig=./capi-aws-quickstart.kubeconfig get nodes
```
```
NAME                                         STATUS   ROLES                  AGE     VERSION
ip-10-0-109-217.eu-west-2.compute.internal   Ready    control-plane,master   8m25s   v1.23.3
ip-10-0-123-211.eu-west-2.compute.internal   Ready    <none>                 7m29s   v1.23.3
ip-10-0-144-233.eu-west-2.compute.internal   Ready    control-plane,master   6m52s   v1.23.3
ip-10-0-241-51.eu-west-2.compute.internal    Ready    control-plane,master   5m17s   v1.23.3
ip-10-0-72-132.eu-west-2.compute.internal    Ready    <none>                 7m29s   v1.23.3
ip-10-0-94-207.eu-west-2.compute.internal    Ready    <none>                 7m29s   v1.23
```
You can access the new cluster using the `kubeconfig` file obtained previously and check that the calico CNI has been installed properly.
```
kubectl --kubeconfig=./capi-aws-quickstart.kubeconfig get pods -A
```

### Instal ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
#### Access ArgoCD API server
By default, the Argo CD API server is not exposed with an external IP. Let's expose it via Port Forwarding. **In a new terminal window** execute the following command.
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

The initial password for the `admin` account is auto-generated and stored as clear text in the field password in a secret named `argocd-initial-admin-secret` in your Argo CD installation namespace.
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
The ArgoCD API server can then be accessed using the `localhost:8080` and `admin` and the `password` from the command above.

