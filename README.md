# CloudFormation Templates for Building OCP

## Table of Contents

<!-- TOC -->
- [CloudFormation Templates for Building OCP](#cloudformation-templates-for-building-ocp)
  - [Table of Contents](#table-of-contents)
  - [IMPORTANT](#important)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
    - [Create Install Config file](#create-install-config-file)
  - [Install Steps](#install-steps)
  - [* use the "ingress" LB and point to *.apps.](#-use-the-ingress-lb-and-point-to-apps)
  - [Additional notes](#additional-notes)
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

To get the CA trust bundle run:

openssl s_client -connect -showcerts <proxy name:port> 

Copy the signing cert into the TrustBundle Above

## Install Steps

1. mkdir install && cp install-config.yaml install
2. openshift-install create manifests --dir=install
3. vi install/manifests/cluster-scheduler-02-config.yml
   1. update schedulable to false
4. edit both `install/manifests/cluster-ingress-02-config.yml` and `cluster-ingress-default-ingresscontroller.yaml`
   1. update endpointPublishingStrategy to be "HostNetwork" 
   BE SURE TO NOT MODIFY YOUR "domain:" entry
```
spec:
  domain: apps.cfbuild.example.com
  endpointPublishingStrategy:
    type: HostNetwork
```
5. openshift-install create ignition-configs --dir=install
6. jq -r .infraID install/metadata.json
   1. update the following files with the cluster name: bootstrap.json(x2), control-plane.json, nw_lb.json, sg_roles.json, worker.json
   2. update step 16 to 18 below with new cluster name in S3 bucket creation step
7. validate settings in nw_lb.json
   1. this should not change for RH testing
8.  cd cf
9.  export AWS_DEFAULT_OUTPUT="text"
10. aws cloudformation create-stack --stack-name cfbuildint-nwlb \
     --template-body file://nw_lb.yaml \
     --parameters file://nw_lb.json \
     --capabilities CAPABILITY_NAMED_IAM
11. aws cloudformation describe-stacks --stack-name cfbuildint-nwlb

-------- External DNS Steps ---------
If you are using an external DNS server you will need to create CNAMES that point to the AWS LB instances that were created
* use the "int" LB and point to both api and api-int
* use the "ingress" LB and point to *.apps.
----------------------------


11. aws cloudformation create-stack --stack-name cfbuildint-sgroles \
     --template-body file://sg_roles.yaml \
     --parameters file://sg_roles.json
12. aws cloudformation describe-stacks --stack-name cfbuildint-sgroles
13. cd ../install
14. aws s3 mb s3://cfbuild-dtxlg-infra
15. aws s3 cp bootstrap.ign s3://cfbuild-dtxlg-infra/bootstrap.ign --acl public-read
16. aws s3 ls s3://cfbuild-dtxlg-infra
17. update bootstrap.json with new S3 bucket
18. cd ../cf
19. update bootstrap.json with the updated SecurityGroups
20. aws cloudformation create-stack --stack-name cfbuildint-bootstrap \
     --template-body file://bootstrap.yaml \
     --parameters file://bootstrap.json

**UPDATE the "int" loadbalancer to have the newly created bootstrap node added here**

21. update control_plane.json with the updated SecurityGroup
22. get the certificate authority from the master.ign file
23. update control_plane.json with the updated certificate authority
24. get a copy of the master.ign from https://api-int.cfbuild.example.com:22623/config/master
25. update master.ign version to 3.1.0
26. upload to s3
27. aws s3 cp master.ign s3://cfbuild-dtxlg-infra/master.ign --acl public-read
28. aws cloudformation create-stack --stack-name cfbuildint-controlplane \
     --template-body file://control-plane.yaml \
     --parameters file://control-plane.json

**UPDATE the "int" loadbalancer to have the newly created control plane nodes added here**

29. cd ..
30. openshift-install wait-for bootstrap-complete --dir=install
31. aws cloudformation delete-stack --stack-name cfbuildint-bootstrap

**UPDATE the "int" loadbalancer and REMOVE the bootstrap node**

32.  get a copy of the master.ign from https://api-int.cfbuild.example.com:22623/config/master
33.  update worker.ign version to 3.1.0
35.  upload to s3
36.  aws s3 cp worker.ign s3://cfbuild-dtxlg-infra/worker.ign --acl public-read
37.  update worker.json with new securtiyGroup ID
38.  update worker.json with new IAM Profile
39.  update CertificateAuthority entry
40.  cd cf
NOTE:  If you want to have your workers on mulitple subnets/AZ be sure to create multiple "worker.json" files and update the subnets for each.
43. aws cloudformation create-stack --stack-name cfbuildint-worker0 \
     --template-body file://worker.yaml \
     --parameters file://worker.json
44. aws cloudformation create-stack --stack-name cfbuildint-worker1 \
     --template-body file://worker.yaml \
     --parameters file://worker.json
45. aws cloudformation create-stack --stack-name cfbuildint-worker2 \
     --template-body file://worker.yaml \
     --parameters file://worker.json
46. cd ..

**UPDATE the "ingress" loadbalancer and ADD the worker nodes created**

47. export KUBECONFIG=$(pwd)/install/auth/kubeconfig
48. oc get nodes
49. oc get csr
50. oc adm certificate approve <csr_name>
51. repeat steps 45 and 50 again (you will need to approve certs 2x for each worker node)
52. openshift-install wait-for install-complete --dir=install


Access Console from:
console-openshift-console.apps.cfbuild.example.com

## Additional notes

It is possible to build using cached copies of the boot info:

{
    "ParameterKey": "IgnitionLocation", 
    "ParameterValue": "s3://cfbuild-dtxlg-infra/worker.ign" 
  },

If DNS does not work, and we need to update DNS servers from the default supplied by AWS, it may be possible to update this at the very beginning.
at step 4 above look at the files in manifest with particular interest around the cluster-dns file and this web page https://docs.openshift.com/container-platform/4.6/networking/dns-operator.html THIS IS UNTESTED DONT USE.

# Cleanup

The following commands will delete your environment ... make sure you really want to do this. You have been warned!

aws cloudformation delete-stack --stack-name cfbuildint-worker0
aws cloudformation delete-stack --stack-name cfbuildint-worker1
aws cloudformation delete-stack --stack-name cfbuildint-worker2
aws cloudformation delete-stack --stack-name cfbuildint-controlplane
aws cloudformation delete-stack --stack-name cfbuildint-bootstrap
