# CloudFormation Templates for Building OCP

## Table of Contents

<!-- TOC -->
- [CloudFormation Templates for Building OCP](#cloudformation-templates-for-building-ocp)
  - [Table of Contents](#table-of-contents)
  - [IMPORTANT](#important)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Installing Your Cluster](#installing-your-cluster)
    - [Create Install Config file](#create-install-config-file)
    - [Install Steps](#install-steps)
  - [Adding Additional Nodes to your cluster](#adding-additional-nodes-to-your-cluster)
  - [Cleanup](#cleanup)
<!-- TOC -->


## IMPORTANT

THIS REPO CONTAINS NO AUTOMATION AROUND AWS Load Balancer configuration. The "nw_lb.yaml" will create two load balancers, as well as the listeners and target groups, but will NOT add any machines to these endpoints. You will need to manually do this during the install.



## Introduction

The following repo documents a process for creating an OpenShift Cluster in AWS on a private VPC minimizing the amount of permissions required to create the cluster. This will create an operational cluster, however it will *NOT* have many of the AWS integrations that are available with the standard IPI install process. This is also a very manual process, and will require editing multiple files to complete. Be sure to have a good editor handy.

## Prerequisites

*Software*
You will need the following software to follow this install process:
* "openshift-install" - https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.6/openshift-install-linux.tar.gz
* "oc" - https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.6/openshift-client-linux.tar.gz
* "aws cli" - https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html

You will need the following information in order to proceed with this install process:
* OpenShift Pull Secret - https://cloud.redhat.com/openshift/install/pull-secret
* SSH Public Key - 
* RHCOS AMI - https://docs.openshift.com/container-platform/4.6/installing/installing_aws/installing-aws-user-infra.html#installation-aws-user-infra-rhcos-ami_installing-aws-user-infra
* AWS VPC ID - this will look something like "vpc-026c4e9f12adc018d"
* AWS Private Subnet ID(s) - this will look something like "subnet-0655610aa6217f120". You may have multiple of these which will help with HA.
* EBS Encryption Key - This is used to encrypt the boot disk at build time. This will look something like "alias/encryptionkeyname"
* AWS Tags - Review the template json and yaml for AWS resource tags that may apply.
* Base Domain name - This is something like example.com
* Cluster Name - This would be the cluster name you are building and will be used to create the fully qualified domain name eg. cfbuild.example.com

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

`openssl s_client -connect -showcerts <proxy name:port>`

Copy the signing cert into the TrustBundle Above

### Install Steps

1. run `mkdir install && cp install-config.yaml install`
2. run `openshift-install create manifests --dir=install`
3. run `vi install/manifests/cluster-scheduler-02-config.yml`
   1. update schedulable to false
4. edit both `install/manifests/cluster-ingress-02-config.yml` and `install/manifests/cluster-ingress-default-ingresscontroller.yaml`
   1. update endpointPublishingStrategy to be "HostNetwork" 
   BE SURE TO NOT MODIFY YOUR "domain:" entry
```
spec:
  domain: apps.cfbuild.example.com
  endpointPublishingStrategy:
    type: HostNetwork
```
5. run `openshift-install create ignition-configs --dir=install`
6. run `jq -r .infraID install/metadata.json`
   1. update the following files with the cluster name: bootstrap.json(x2), control-plane.json, nw_lb.json, sg_roles.json, worker.json
   2. update step 16 to 18 below with new cluster name in S3 bucket creation step
7. validate settings in nw_lb.json
8.  run the following commands:
```
$ cd cf
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

9.  Create the security groups with
```
$ aws cloudformation create-stack --stack-name cfbuildint-sgroles \
     --template-body file://sg_roles.yaml \
     --parameters file://sg_roles.json`
```
10. run `aws cloudformation describe-stacks --stack-name cfbuildint-sgroles`
11. Create place the bootstrap ignition file somewhere it can be accessed by the bootstrap node:
```
$ cd ../install
$ aws s3 mb s3://cfbuild-2dpcg-infra
$ aws s3 cp bootstrap.ign s3://cfbuild-2dpcg-infra/bootstrap.ign --acl public-read
$ aws s3 ls s3://cfbuild-2dpcg-infra
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

16. update control_plane.json with the updated SecurityGroup
17. get the certificate authority from the master.ign file
18. update control_plane.json with the updated certificate authority
19. run the following command to create the control-plane:
```
aws cloudformation create-stack --stack-name cfbuildint-controlplane /\
     --template-body file://control-plane.yaml /\
     --parameters file://control-plane.json
```

**Manually UPDATE the "int" loadbalancer to have the newly created control plane nodes added here**

20. run `cd ..`
21. run the following command to watch the initial bootstrap
`openshift-install wait-for bootstrap-complete --dir=install`
22. When the initial install is complete run the following command to delete the bootstrap node:
`aws cloudformation delete-stack --stack-name cfbuildint-bootstrap`

**Manually UPDATE the "int" loadbalancer and REMOVE the bootstrap node**

23. update worker.json with new securtiyGroup ID
24. update worker.json with new IAM Profile
25. update CertificateAuthority entry
26. run `cd cftemplates`
NOTE:  If you want to have your workers on mulitple subnets/AZ be sure to create multiple "worker.json" files and update the subnets for each.
27.  Create as many worker nodes as you will require for your cluster. Note that you must create a MINIMUM of three worker nodes:
```
$ aws cloudformation create-stack --stack-name cfbuildint-worker0 /\
     --template-body file://worker.yaml /\
     --parameters file://worker.json
$ aws cloudformation create-stack --stack-name cfbuildint-worker1 /\
     --template-body file://worker.yaml /\
     --parameters file://worker.json
$ aws cloudformation create-stack --stack-name cfbuildint-worker2 /\
     --template-body file://worker.yaml /\
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


## Cleanup

The following commands will delete your environment ... make sure you really want to do this. You have been warned!

aws cloudformation delete-stack --stack-name cfbuildint-worker0
aws cloudformation delete-stack --stack-name cfbuildint-worker1
aws cloudformation delete-stack --stack-name cfbuildint-worker2
aws cloudformation delete-stack --stack-name cfbuildint-controlplane
aws cloudformation delete-stack --stack-name cfbuildint-bootstrap