# CloudFormation Templates for Building OCP

## Table of Contents

<!-- TOC -->
- [CloudFormation Templates for Building OCP](#cloudformation-templates-for-building-ocp)
  - [Table of Contents](#table-of-contents)
  - [IMPORTANT](#important)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
    - [DNS Requirements](#dns-requirements)
  - [Installing Your Cluster](#installing-your-cluster)
    - [Create Install Config file](#create-install-config-file)
    - [Install Steps](#install-steps)
  - [Adding Additional Nodes to your cluster](#adding-additional-nodes-to-your-cluster)
  - [Removing Worker Nodes](#removing-worker-nodes)
  - [Replacing Worker Nodes](#replacing-worker-nodes)
  - [Replacing Master Nodes](#replacing-master-nodes)
    - [Identifying the leader node](#identifying-the-leader-node)
    - [Clean Out old Node Entries](#clean-out-old-node-entries)
      - [Cleanup old etcd entries](#cleanup-old-etcd-entries)
      - [Cleanup old secrets](#cleanup-old-secrets)
      - [Clean up AWS Target Groups](#clean-up-aws-target-groups)
    - [Building a new master node](#building-a-new-master-node)
    - [Updating AWS Loadbalancer TargetGroups](#updating-aws-loadbalancer-targetgroups)
  - [Cleanup](#cleanup)
<!-- TOC -->

## IMPORTANT

THIS REPO CONTAINS NO AUTOMATION AROUND AWS Load Balancer configuration. The "nw_lb.yaml" will create two load balancers, as well as the listeners and target groups, but will NOT add any machines to these endpoints. You will need to manually do this during the install.

## Introduction

The following repo documents a process for creating an OpenShift Cluster in AWS on a private VPC minimizing the amount of permissions required to create the cluster. This will create an operational cluster, however it will *NOT* have many of the AWS integrations that are available with the standard IPI install process. This install leverages only one AWS subnet, and thus should not be used for production workloads. This is also a very manual process, and will require editing multiple files to complete. Be sure to have a good editor handy.

## Prerequisites

*Software*
You will need the following software to follow this install process:
* "openshift-install" - https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.6/openshift-install-linux.tar.gz
* "oc" - https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.6/openshift-client-linux.tar.gz
* "aws cli" - https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html

You will need the following information in order to proceed with this install process:
* OpenShift Pull Secret - https://cloud.redhat.com/openshift/install/pull-secret
* SSH Public Key - https://docs.openshift.com/container-platform/4.6/installing/installing_aws/installing-aws-default.html#ssh-agent-using_installing-aws-default
* RHCOS AMI - https://docs.openshift.com/container-platform/4.6/installing/installing_aws/installing-aws-user-infra.html#installation-aws-user-infra-rhcos-ami_installing-aws-user-infra
* AWS VPC ID - this will look something like "vpc-026c4e9f12adc018d"
* AWS Private Subnet ID - this will look something like "subnet-0655610aa6217f120".
* EBS Encryption Key - This is used to encrypt the boot disk at build time. This will look something like "alias/encryptionkeyname"
* AWS Tags - Review the template json and yaml for AWS resource tags that may apply.
* Base Domain name - This is something like example.com
* Cluster Name - This would be the cluster name you are building and will be used to create the fully qualified domain name eg. cfbuild.example.com
* S3 Bucket OR a separate web server that supports https with a valid signed certificate that can host the bootstrap.ign file
* AWS credentials, or an AWS ec2 instance with the appropriate roles assigned for the creation of EC2 instances, as well as load balancers and security groups

### DNS Requirements

You will need three DNS entries for your cluster:

1. api.\<clustername\>.\<basename\>
2. api-int.\<clustername\>.\<basename\>
3. *.apps.\<clustername\>.\<basename\>

Where \<clustername\> is the name of your OpenShift cluster such as "ocp47", and \<basename\> is the base domain name such as "test.example.com". The entries for "api." and "api-int." will both point to one Network LoadBalancer that we will create later, and the "*.apps." will point to the second network load balancer. 

## Installing Your Cluster

### Create Install Config file

Leveraging the `install-config.yaml` template in the root directory of this repo, update the following fields:
* baseDomain
* metadata.name
* pullSecret
* sshKey

Review the "networking" section and ensure that the IP address ranges do not conflict in your network. You will also need to update networking.machineNetwork.cidr to match the network you are deploying your machines on.

*Proxy Config*
If you need to use Proxy to access outside resources, we will need to update the install-config.yaml file with a proxy section as listed below:
Add the following to the install-config.yaml

```
proxy:
  httpProxy: http://<username>:<pswd>@<ip>:<port> 
  httpsProxy: http://<username>:<pswd>@<ip>:<port> 
  noProxy: example.com 
additionalTrustBundle: | 
    -----BEGIN CERTIFICATE-----
    <MY_TRUSTED_CA_CERT>
    -----END CERTIFICATE-----
```

If your proxy has a self signed certificate, you will need to get a copy of that certificate and add it to the "additionalTrustBundle" section called out above.

To get the CA trust bundle run:

`openssl s_client -connect -showcerts \<proxy name:port\>`

Copy the signing cert into the TrustBundle Above

### Install Steps

1. run `mkdir install && cp install-config.yaml install`
2. run `openshift-install create manifests --dir=install`
3. run `vi install/manifests/cluster-scheduler-02-config.yml`
   1. update schedulable to false
4. edit both `install/manifests/cluster-ingress-02-config.yml` and `install/manifests/cluster-ingress-default-ingresscontroller.yaml`
   1. update endpointPublishingStrategy to be "HostNetwork" 
   BE SURE TO NOT MODIFY YOUR "domain:" entry

cluster-ingress-02-config.yaml
```
spec:
  domain: apps.cfbuild.example.com
  endpointPublishingStrategy:
    type: HostNetwork
```

cluster-ingress-default-ingresscontroller.yaml
```
spec:
  endpointPublishingStrategy:
    loadBalancer:
      scope: Internal
    type: HostNetwork
```

5. run `openshift-install create ignition-configs --dir=install`
6. run `jq -r .infraID install/metadata.json`
   1. update the following files with the cluster name: bootstrap.json(x2), control-plane.json, nw_lb.json, sg_roles.json, worker.json
   2. update step 11 below with new cluster name in S3 bucket creation step
7. validate settings in nw_lb.json ensuring you update all placeholders
8.  run the following commands:
```
$ cd cftemplates
$ export AWS_DEFAULT_OUTPUT="text"
$ aws cloudformation create-stack --stack-name cfbuildint-nwlb \
     --template-body file://nw_lb.yaml \
     --parameters file://nw_lb.json 
$ aws cloudformation describe-stacks --stack-name cfbuildint-nwlb
```

>**External DNS Steps**
>If you are using an external DNS server you will need to create CNAMES that point to the AWS LB instances that were created
>  * use the "int" LB and point to both api and api-int
>  * use the "ingress" LB and point to *.apps.

9.  Update sg_roles.json with appropriate values then create the security groups with
```
$ aws cloudformation create-stack --stack-name cfbuildint-sgroles \
     --template-body file://sg_roles.yaml \
     --parameters file://sg_roles.json
```
10. run `aws cloudformation describe-stacks --stack-name cfbuildint-sgroles`
11. Place the bootstrap ignition file somewhere it can be accessed by the bootstrap node:
```
$ cd ../install
$ aws s3 mb s3://aws47-bp45h-infra
$ aws s3 cp bootstrap.ign s3://aws47-bp45h-infra/bootstrap.ign --acl public-read
$ aws s3 ls s3://aws47-bp45h-infra
```
12. update bootstrap.json with new S3 bucket

> **NOTE**
> If you are unable to use S3 buckets for storing the bootstrap.ign file, you can host the file on a web server instead. The webserver _MUST_ support HTTPS and
> the SSL Certificate _MUST_ be signed by a trusted CA for the bootstrap process to proceed. Be sure to update the "BootstrapIgnitionLocation" in the bootstrap.json file
> with the proper URL to access the bootstrap.ign file (eg. "https://someserver.somedomain.com/bootstrap.ign" ).

13. run `cd ../cftemplates`
14. update bootstrap.json with the updated SecurityGroups
15. run the following command to create the bootstrap node:
```
aws cloudformation create-stack --stack-name cfbuildint-bootstrap \
     --template-body file://bootstrap.yaml \
     --parameters file://bootstrap.json
```
**Manually UPDATE the "int" loadbalancer to have the newly created bootstrap node added here**
     a. from AWS Console, select EC2->Load Balancers->Target Groups
     b. find your int 6443 target, add the bootstrap node ip address
     c. find your int 22623 target, add the bootstrap node ip address

1.  update control-plane.json with the updated SecurityGroup
2.  get the certificate authority from the master.ign file
3.  update control-plane.json with the updated certificate authority
4.  update control-plane.json with the updated IgnitionLocation
5.  run the following command to create the control-plane:
```
aws cloudformation create-stack --stack-name cfbuildint-controlplane \
     --template-body file://control-plane.yaml \
     --parameters file://control-plane.json
```

**Manually UPDATE the "int" loadbalancer to have the newly created control plane nodes added here**
     a. from AWS Console, select EC2->Load Balancers->Target Groups
     b. find your int 6443 target, add the three new control plane nodes ip addresses
     c. find your int 22623 target, add the three new control plane nodes ip addresses

1.  run `cd ..`
2.  run the following command to watch the initial bootstrap
`openshift-install wait-for bootstrap-complete --dir=install`
22. When the initial install is complete run the following command to delete the bootstrap node:
`aws cloudformation delete-stack --stack-name cfbuildint-bootstrap`

**Manually UPDATE the "int" loadbalancer and REMOVE the bootstrap node**

23. update worker.json with new securtiyGroup ID
24. update CertificateAuthority entry
25. update IgnitionLocation entry
26. run `cd cftemplates`
NOTE:  If you want to have your workers on mulitple subnets/AZ be sure to create multiple "worker.json" files and update the subnets for each.
27.  Create as many worker nodes as you will require for your cluster. Note that you must create a MINIMUM of three worker nodes:
```
$ aws cloudformation create-stack --stack-name cfbuildint-worker0 \
     --template-body file://worker.yaml \
     --parameters file://worker.json
$ aws cloudformation create-stack --stack-name cfbuildint-worker1 \
     --template-body file://worker.yaml \
     --parameters file://worker.json
$ aws cloudformation create-stack --stack-name cfbuildint-worker2 \
     --template-body file://worker.yaml \
     --parameters file://worker.json
```
28. run `cd ..`

**Manually UPDATE the "ingress" loadbalancer and ADD the worker nodes created**

29. run `export KUBECONFIG=$(pwd)/install/auth/kubeconfig`
30. run `oc get nodes`
31. run `oc get csr`
32. run `oc adm certificate approve <csr_name>`
33. repeat steps 45 and 50 again (you will need to approve certs 2x for each worker node)
34. run `openshift-install wait-for install-complete --dir=install`

Access Console from:
console-openshift-console.apps.cfbuild.example.com

## Adding Additional Nodes to your cluster

When building your cluster in the above described manner you can add additional worker nodes up to the cluster using your existing cftemplates/worker.json file for up to 24 hours. After 24 hours, the machineConfig controller creates a new signing certificate and any NEW EC2 instance created will no longer trust the cluster for initial build.  In order to add additional worker nodes to your cluster after the 24 hour mark, you will need to follow a few additional steps to get the new signing certificate and add it to your existing configuration files.  **NOTE** These steps assume that you have your original "worker.json" file from when you first built your cluster. If you do not, you will need to follow the steps in the [Install Steps](#install-steps) above to re-create your worker.json file.

First step is to get the new machineConfig signing certificate:

1. get the value of "IgnitionLocation" from the cftemplates/worker.json file.
2. run the following command updating the export command with the hostname and port from step one
```
$ export MCS=api-int.clusterDomain:22623
$ echo "q" | openssl s_client -connect $MCS  -showcerts | awk '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/' | base64 --wrap=0 > ./api-int.base64
$ cat api-int.base64
```
3. update your cftemplates/worker.json file with the output from step 2. specifically you will want to update the base64 encoded string for "CertificateAuthorities"
```
},
  {
    "ParameterKey": "CertificateAuthorities", 
    "ParameterValue": "data:text/plain;charset=utf-8;base64,<insert update string here>" 
  },
```
4. Run the following command to create one or more additional nodes (be sure to update the stack name to something that has not been used):
```
aws cloudformation create-stack --stack-name <new stack name> \
     --template-body file://worker.yaml \
     --parameters file://worker.json
```
5. You can repeat step #4 as many times as you want to add additional worker nodes to your cluster
6. log into your cluster with a user with cluster admin access
7. oc get csr 
8. oc adm certificate approve <csr_name>
9. repeat steps 7 and 8 again (you will need to approve certs 2x for each worker node you add)

## Removing Worker Nodes

To replace a worker node, add a new node following the instructions listed in [Adding Additional Nodes to your cluster](#adding-additional-nodes-to-your-cluster). Once your additional node(s) have been added to your cluster you can remove any old nodes you no longer want in your cluster.  Follow these steps:

```
$ oc get nodes
$ oc adm cordon <node_name>
$ oc adm drain <node_name> --force --delete-local-data --ignore-daemonsets
$ oc delete node <node_name>
```

At this point you should terminate the EC2 instance that you removed from your cluster.

## Replacing Worker Nodes

To replace worker nodes within your cluster leverage the steps outlined in [Adding Additional Nodes to your Cluster](#adding-additional-nodes-to-your-cluster) followed by [Removing Worker Nodes](#removing-worker-nodes).

**Manually UPDATE the "ingress" loadbalancer and ADD the worker nodes created**
Ensure that as you replace the worker nodes you update the AWS Loadbalancer instance adding and removing the worker nodes as you proceed. If you do not do this, you will loose access to the console and all applications hosted at \*.apps.\<cluster name\>

Log into your AWS console and update the following EC2 Target Groups adding the new node to the target group.

- \<clusterName\>-ingress-\<random GUID\> port 443
- \<clusterName\>-ingress-\<random GUID\> port 80

## Replacing Master Nodes

The following instructions will detail how to replace a failed Master node. If you are looking to replace a working, running node, you will need to delete that node first prior to starting these steps.

**NOTE:**  You are effecting the redundancy and high availability of the cluster when you delete a master node. Be sure you know what you are doing before proceeding with these steps. **You have been warned**.

Before proceeding with these steps it is recommended that you take a backup of your etcd database. See [Backing up etcd](https://docs.openshift.com/container-platform/4.8/backup_and_restore/control_plane_backup_and_restore/backing-up-etcd.html) for full steps.

If your master node has already failed, you can jump to [Clean Out Old Node Entries](#clean-out-old-node-entries). If you are manually removing a working node, it is recommended that you first identify a non-leader node to remove, and only remove the leader node if you must.

### Identifying the leader node

To identify the leader and non-leader nodes, we will need to connect to one of the running etcd pods and use the *etcdctl* command to get the node status.

```
$ oc project openshift-etcd
$ oc get po
NAME                                READY   STATUS      RESTARTS   AGE
etcd-ip-10-0-56-139                 4/4     Running     0          39h
etcd-ip-10-0-56-14                  4/4     Running     0          39h
etcd-ip-10-0-60-68                  4/4     Running     0          39h
...
revision-pruner-3-ip-10-0-56-14     0/1     Completed   0          39h
revision-pruner-3-ip-10-0-60-68     0/1     Completed   0          39h
$ oc rsh etcd-ip-10-0-56-139
Defaulted container "etcdctl" out of: etcdctl, etcd, etcd-metrics, etcd-health-monitor, setup (init), etcd-ensure-env-vars (init), etcd-resources-copy (init)
sh-4.4# etcdctl endpoint status -w table
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.0.56.139:2379 | f5d95a9b9bd5f0a1 |  3.4.14 |   86 MB |      true |      false |         6 |    1058737 |            1058737 |        |
|  https://10.0.56.14:2379 |  83091a4e8a1b2f2 |  3.4.14 |   86 MB |     false |      false |         6 |    1058737 |            1058737 |        |
|  https://10.0.60.68:2379 | 5735e4d448e73db3 |  3.4.14 |   86 MB |     false |      false |         6 |    1058737 |            1058737 |        |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

In the above output 10.0.56.139 is the Leader.

### Clean Out old Node Entries

Start by deleting/terminating the node from AWS if it still exists.

Now remove the failed node from the OpenShift cluster:

```
$ oc get nodes
NAME             STATUS     ROLES    AGE   VERSION
ip-10-0-56-139   Ready      master   40h   v1.21.6+c180a7c
ip-10-0-56-14    NotReady   master   40h   v1.21.6+c180a7c
ip-10-0-57-243   Ready      worker   39h   v1.21.6+c180a7c
ip-10-0-60-68    Ready      master   40h   v1.21.6+c180a7c
ip-10-0-61-216   Ready      worker   39h   v1.21.6+c180a7c
ip-10-0-62-138   Ready      worker   39h   v1.21.6+c180a7c
$ oc delete node/<node-name>
$ oc get nodes
NAME             STATUS   ROLES    AGE   VERSION
ip-10-0-56-139   Ready    master   40h   v1.21.6+c180a7c
ip-10-0-57-243   Ready    worker   40h   v1.21.6+c180a7c
ip-10-0-60-68    Ready    master   40h   v1.21.6+c180a7c
ip-10-0-61-216   Ready    worker   40h   v1.21.6+c180a7c
ip-10-0-62-138   Ready    worker   40h   v1.21.6+c180a7c
$ oc delete machine/<machine-name>
```

#### Cleanup old etcd entries

We now need to cleanup and remove all old etcd entries for the failed node. In the examples below the node that was removed is called "ip-10-0-56-14", we will be removing the entries for this node. Be sure to update the commands for your specific cluster.

```
$ oc project openshift-etcd
$ oc get po
NAME                                READY   STATUS      RESTARTS   AGE
etcd-ip-10-0-56-139                 4/4     Running     0          39h
etcd-ip-10-0-60-68                  4/4     Running     0          39h
...
revision-pruner-3-ip-10-0-60-68     0/1     Completed   0          39h
$ oc rsh etcd-ip-10-0-56-139
$ etcdctl member list -w table
+------------------+---------+----------------+--------------------------+--------------------------+------------+
|        ID        | STATUS  |      NAME      |        PEER ADDRS        |       CLIENT ADDRS       | IS LEARNER |
+------------------+---------+----------------+--------------------------+--------------------------+------------+
|  83091a4e8a1b2f2 | started |  ip-10-0-56-14 |  https://10.0.56.14:2380 |  https://10.0.56.14:2379 |      false |
| 5735e4d448e73db3 | started |  ip-10-0-60-68 |  https://10.0.60.68:2380 |  https://10.0.60.68:2379 |      false |
| f5d95a9b9bd5f0a1 | started | ip-10-0-56-139 | https://10.0.56.139:2380 | https://10.0.56.139:2379 |      false |
+------------------+---------+----------------+--------------------------+--------------------------+------------+
$ etcdctl member remove   83091a4e8a1b2f2
$ etcdctl member list -w table
+------------------+---------+----------------+--------------------------+--------------------------+------------+
|        ID        | STATUS  |      NAME      |        PEER ADDRS        |       CLIENT ADDRS       | IS LEARNER |
+------------------+---------+----------------+--------------------------+--------------------------+------------+
| 5735e4d448e73db3 | started |  ip-10-0-60-68 |  https://10.0.60.68:2380 |  https://10.0.60.68:2379 |      false |
| f5d95a9b9bd5f0a1 | started | ip-10-0-56-139 | https://10.0.56.139:2380 | https://10.0.56.139:2379 |      false |
+------------------+---------+----------------+--------------------------+--------------------------+------------+
```

**NOTE**: After deleting a master node, ETCD may be going through a re-election and you may get errors from the etcdctl commands, just re-run them until they work.

#### Cleanup old secrets

Now lets cleanup the secrets. Be sure to update the delete commands to reflect the names of your cluster's secrets

```
$ oc get secrets -n openshift-etcd | grep ip-10-0-56-14
etcd-peer-ip-10-0-56-14               kubernetes.io/tls                     2      41h
etcd-serving-ip-10-0-56-14            kubernetes.io/tls                     2      41h
etcd-serving-metrics-ip-10-0-56-14    kubernetes.io/tls                     2      41h
$ oc delete secret etcd-peer-ip-10-0-56-14 
$ oc delete secret etcd-serving-ip-10-0-56-14
$ oc delete secret etcd-serving-metrics-ip-10-0-56-14
```

#### Clean up AWS Target Groups

You should remove the failed master node from the AWS Target groups at this time. The following target groups will need to be updated to have the failed node removed:

- \<clusterName\>-Exter-\<random GUID\> port 6443
- \<clusterName\>-Inter-\<random GUID\> port 22623
- \<clusterName\>-Inter-\<random GUID\> port 6443

### Building a new master node

Using the **cftemplates\control_plane_single_node.json** file, update all the values in that document. You should be able to use your original "control-plane.json" file as a template. You will need to update the certificate that is in the file.

First step is to get the new machineConfig signing certificate:

1. get the value of "IgnitionLocation" from the cftemplates/control_plane_single_node.json file.
2. run the following command updating the export command with the hostname and port from step one
```
$ export MCS=api-int.clusterDomain:22623
$ echo "q" | openssl s_client -connect $MCS  -showcerts | awk '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/' | base64 --wrap=0 > ./api-int.base64
$ cat api-int.base64
```
3. update your cftemplates/control_plane_single_node.json file with the output from step 2. specifically you will want to update the base64 encoded string for "CertificateAuthorities"
```
},
  {
    "ParameterKey": "CertificateAuthorities", 
    "ParameterValue": "data:text/plain;charset=utf-8;base64,<insert update string here>" 
  },
```

After validating all the entries in your **cftemplates\control_plane_single_node.json**, run the following command to create a new master node (be sure to update the stack name to something that has not been used):

```
aws cloudformation create-stack --stack-name replacementMaster0 \
     --template-body file://control_plane_single_node.yaml \
     --parameters file://control_plane_single_node.json
```

The new master node should start to be deployed it will take 5-10 minutes for it to first attempt to join the cluster. You will not be able to proceed to the next step until it has started the cluster join process.

1. log into your cluster with a user with cluster admin access
2. oc get csr
If you do not get any Pending csr requests, you may need to wait longer
3. oc get csr -o name | xargs oc adm certificate approve
4. repeat steps 7 and 8 again (you will need to approve certs 2x for each worker node you add)

At this point, your new master node is built and the etcd quorum should start to rebuild on its own. This will take some time. We can validate the process is complete with the following command:

```
$ oc get pods -n openshift-etcd| grep -v etcd-quorum-guard|grep etcd
etcd-ip-10-0-56-139                 4/4     Running     0          12m
etcd-ip-10-0-60-68                  4/4     Running     0          11m
etcd-ip-10-0-62-245                 4/4     Running     5          17m
```

NOTE:  During the etcd rebuild time you may experience Kubernetes API errors, this should be expected during the rebuild process.

We can also double check the state of the etcd rebuild by connecting to the new etcd instance and running a few etcdctl commands:

```
$ oc project openshift-etcd
$ oc get po
NAME                                READY   STATUS      RESTARTS   AGE
etcd-ip-10-0-56-139                 4/4     Running     0          39h
etcd-ip-10-0-56-14                  4/4     Running     0          39h
etcd-ip-10-0-62-245                 4/4     Running     5          17m
...
revision-pruner-3-ip-10-0-56-14     0/1     Completed   0          39h
revision-pruner-3-ip-10-0-60-68     0/1     Completed   0          39h
$ oc rsh etcd-ip-10-0-62-245
Defaulted container "etcdctl" out of: etcdctl, etcd, etcd-metrics, etcd-health-monitor, setup (init), etcd-ensure-env-vars (init), etcd-resources-copy (init)
sh-4.4# etcdctl endpoint status -w table
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.0.56.139:2379 | f5d95a9b9bd5f0a1 |  3.4.14 |   86 MB |     false |      false |       127 |    1098530 |            1098530 |        |
|  https://10.0.60.68:2379 | 5735e4d448e73db3 |  3.4.14 |   86 MB |     false |      false |       127 |    1098530 |            1098530 |        |
| https://10.0.62.245:2379 | bc83e15406fede8d |  3.4.14 |   86 MB |      true |      false |       127 |    1098530 |            1098530 |        |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

### Updating AWS Loadbalancer TargetGroups

At this point you will need to manually update target groups for you AWS Load Balancer. Log into your AWS console and update the following EC2 Target Groups:

- \<clusterName\>-Exter-\<random GUID\> port 6443
- \<clusterName\>-Inter-\<random GUID\> port 22623
- \<clusterName\>-Inter-\<random GUID\> port 6443

Be sure the old master node IP has been removed from the list, and add the new node's IP to the list.

**NOTE:  DO NOT** replace additional master nodes, until the newly added node shows as active within the AWS TargetGroup. After replacing a master node, it may take some time for the kubeapi to settle, ensure ample time for the cluster to settle.

## Cleanup

The following commands will delete your environment ... make sure you really want to do this. You have been warned!

aws cloudformation delete-stack --stack-name cfbuildint-worker0
aws cloudformation delete-stack --stack-name cfbuildint-worker1
aws cloudformation delete-stack --stack-name cfbuildint-worker2
aws cloudformation delete-stack --stack-name cfbuildint-controlplane
aws cloudformation delete-stack --stack-name cfbuildint-bootstrap
aws cloudformation delete-stack --stack-name cfbuildint-sgroles
aws cloudformation delete-stack --stack-name cfbuildint-nwlb
