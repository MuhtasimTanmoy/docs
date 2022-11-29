# How to create a Kubernetes cluster for Testground

_Testground is not coupled with AWS, but the infrastructure playbooks described in this section are targeted at AWS._

## Requirements

First and foremost, you need an AWS account with API access.

Next, download and install all required software:

1. [kops](https://github.com/kubernetes/kops/releases) &gt;= 1.17.0
2. [terraform](https://terraform.io/) &gt;= 0.12.21
3. [AWS CLI](https://aws.amazon.com/cli)
4. [helm](https://github.com/helm/helm) &gt;= 3.0
5. [kubectl](https://kubernetes.io/docs/tasks/tools/)

## Setup AWS cloud credentials

1. [Generate your AWS IAM credentials](https://console.aws.amazon.com/iam/home#/security_credentials).
2. [Configure the aws-cli tool with your credentials](https://docs.aws.amazon.com/cli/).

## Generate a Testground SSH key for `kops`

It is used for the Kubernetes master and worker nodes

```bash
# generate ~/.ssh/testground_rsa
#          ~/.ssh/testground_rsa.pub

$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com" \
                            -f ~/.ssh/testground_rsa -q -P ""
```

## Create a bucket for `kops` state

This is similar to Terraform state bucket.

```text
$ aws s3api create-bucket \
      --bucket <bucket_name> \
      --region <region> --create-bucket-configuration LocationConstraint=<region>
```

Where:

* `<bucket_name>` is an AWS account-wide unique bucket name to store this cluster's `kops` state, e.g. `kops-backend-bucket-<your_username>`.
* `<region>` is an AWS region like `eu-central-1` or `us-west-2`.

## Configure cluster variables

* a cluster name \(for example `name.k8s.local`\)
* set AWS region
* set AWS availability zone A \(not region; for example `us-west-2a` \[availability zone\]\) - used for master node and worker nodes
* set AWS availability zone B \(not region; for example `us-west-2b` \[availability zone\]\) - used for more worker nodes
* set `kops` state store bucket \(the bucket we created in the section above\)
* set number of worker nodes
* set master node instance type \(read on best practices at [https://kubernetes.io/docs/setup/best-practices/cluster-large/\#size-of-master-and-master-components](https://kubernetes.io/docs/setup/best-practices/cluster-large/#size-of-master-and-master-components)\)
* set worker node instance type
* set location of your cluster SSH public key \(for example `~/.ssh/testground_rsa.pub` generated above\)
* set team and project name - these values are used as tags in AWS for cost allocation purposes

You might want to add them to your `rc` file \(`.zshrc`, `.bashrc`, etc.\), or to an `.env.sh` file that you source.

```bash
export CLUSTER_NAME=<desired kubernetes cluster name (e.g. mycluster.k8s.local)>
export DEPLOYMENT_NAME=<desired kubernetes deployment name>
export KOPS_STATE_STORE=s3://<kops state s3 bucket>
export AWS_REGION=<aws region, for example eu-central-1>
export ZONE_A=<aws availability zone, for example eu-central-1a>
export ZONE_B=<aws availability zone, for example eu-central-1b>
export WORKER_NODES=4
export MASTER_NODE_TYPE=c5.2xlarge
export WORKER_NODE_TYPE=c5.2xlarge
export PUBKEY=$HOME/.ssh/testground_rsa.pub
export TEAM=<your team name ; tag is used for cost allocation purposes>
export PROJECT=<your project name ; tag is used for cost allocation purposes>
```

## Setup required Helm chart repositories

```text
$ helm repo add stable https://charts.helm.sh/stable
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm repo add influxdata https://helm.influxdata.com/
$ helm repo update
```

## Configure the Testground client

Create a `.env.toml` file in your `$TESTGROUND_HOME` and add your AWS region to the `["aws"]` section, as well as the correct endpoint.
The endpoint refers to the `testground-daemon` service, so depending on your setup, this could be, for example, a Load Balancer fronting the kubernetes cluster and forwarding proper requests to the `tg-daemon` service, or a simple port forward to your local workstation:

```
["aws"]
region = "eu-central-1"  # edit to match your region
[client]
endpoint = "http://localhost:28015" # in case we use port forwarding, like this one here: kubectl port-forward service/testground-daemon 28015:8042
```

## Create cloud resources for the Kubernetes cluster

This will take about 10-15 minutes to complete.

Once you run this command, take some time to walk the dog, clean up around the office, or go get yourself some coffee! When you return, your shiny new Kubernetes cluster will be ready to run Testground plans.

```bash
$ git clone https://github.com/testground/infra

$ cd infra/k8s

# Install AWS cloud resources with `kops`
$ ./01_install_k8s.sh cluster.yaml

# Install EFS file system and mount it to the cluster
$ ./02_efs.sh cluster.yaml

# Add EBS storage class to the cluster
$ ./03_ebs.sh cluster.yaml

# Install the remote Testground daemon
$ ./04_testground_daemon.sh cluster.yaml
```

## Destroy the cluster when you're done working on it

Do not forget to delete the cluster once you are done running test plans.

```bash
$ cd infra/k8s

# Remove EBS volumes
$ ./delete_ebs.sh

# Remove EFS file system
$ ./delete_efs.sh

# Remove AWS cloud resources created by `kops` - EC2 VMs, security groups,
# auto-scaling groups, etc.
$ ./delete_kops.sh
```

## Resizing the cluster

Edit the cluster state and change number of nodes.

```text
$ kops edit ig nodes
```

Apply the new configuration

```text
$ kops update cluster $NAME --yes
```

Wait for nodes to come up and for `DaemonSets` to be `Running` on all new nodes

```text
$ watch 'kubectl get pods'
```

## Use a Kubernetes context for another cluster

`kops` lets you download the entire Kubernetes context config.

If you want to let other people on your team connect to your Kubernetes cluster, you need to give them the information.

```text
$ kops export kubecfg --state $KOPS_STATE_STORE --name=$NAME
```
