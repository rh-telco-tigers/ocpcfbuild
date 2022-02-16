# Installing AWS with Cloud Formation Templates and STS Auth

## Why do this?

Currently the OpenShift installer requires the use of Route53 DNS to build a cluster. If you are not a user of Route53 but still want to build an OpenShift cluster in AWS and you want to leverage the more secure STS tokens for cluster integration, this post is for you. This post will show how to create an OpenShift deployment with no Route53 or AWS Lambda functions. You can enable these additional functions if you would like to ease the deployment.

## Prerequisites

- Linux host to runs commands from
- Existing VPC
- Existing Subnet
- External DNS that you will configure
- openshift-install command
- oc command

## Configure Security Groups

Using the template file *cftemplates-sts/sg_roles.json* update the file with the values for your existing VPC and subnet. Apply the sg_roles CloudFormation template to setup the security groups and IAM roles.

```shell
$ aws cloudformation create-stack --stack-name awssts-sgroles \
     --template-body file://sg_roles.yaml \
     --parameters file://sg_roles.json \
     --capabilities CAPABILITY_NAMED_IAM
$ aws cloudformation describe-stacks --stack-name awssts-sgroles
```

Take note  of the output as you will need the security group information later on.

## Get the ccoctl utility

The ccoctl utility will be used to help set up STS tokens in your account.

> **NOTE**: You will need to run these commands from a Linux host. The ccoctl utility is a Linux only command and will not work on Windows or Mac.

You will need to have your pull secret in order to proceed. Before proceeding, ensure you have downloaded a copy of your [Pull Secret](https://console.redhat.com/openshift/install/pull-secret) and stored it in a file called `~/.pull-secret`. You can also store the file in another location, but be sure to update the commands below to point to the proper file.

```shell
$ export VERSION=stable-4.9
export RELEASE_IMAGE=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/release.txt | grep 'Pull From: quay.io' | awk -F ' ' '{print $3}')
CCO_IMAGE=$(oc adm release info --image-for='cloud-credential-operator' $RELEASE_IMAGE)
oc image extract $CCO_IMAGE --file="/usr/bin/ccoctl" -a ~/.pull-secret
chmod 775 ccoctl
```

Check and ensure that the utility is working:

```shell
$ ./ccoctl aws --help
```

### Extract the Credentials

Be sure to update \<name\> with the name of the cluster and \<aws_region\> with the region you are working in

```shell
$ mkdir -p stsinstall/credrequests
$ cd stsinstall
$ oc adm release extract --credentials-requests --cloud=aws --to=$(pwd)/credrequests $RELEASE_IMAGE
$ ccoctl aws create-all --name=<name> --region=<aws_region> --credentials-requests-dir=$(pwd)/credrequests
```


## Create the install-config file

Create an install-config.yaml by starting with the *install-config-awssts.yaml* as a template. Be sure to update the following sections:

- baseDomain
- metadata.name
- networking.machineNetwork
- platform.aws.region
- platform.aws.userTags
- pullSecret
- sshKey

With an updated install-config.yaml file,  we will use the OpenShift install utility to create a base configuration that we will modify to work with STS tokens.

Start by making a directory called "install" and placing a copy of the install-config.yaml file in that directory:

```shell
mkdir install
cp install-config.yaml install
```

> **NOTE:** It is suggested that you make a _copy_ of the file, as it will be automatically deleted later in the process.

Next we will create the OpenShift manifest files that define the new cluster.

```shell
openshift-install create manifests --dir=install
```

You will now have an _install_ directory that contains the files that will define your cluster. We will copy in the STS files that we created as part of the STS configuration into the directory structure.

```shell
cp stsinstall/manifests/* install/manifests/
cp -a stsinstall/tls install/
```

With the STS tokens copied into our build directory, we now need to update additional files in the directory. First we need to delete some file that will not be required.

```shell
rm install/openshift/99_openshift-cluster-api_master-*.yaml
rm install/openshift/99_openshift-cluster-api_worker-machineset-1.yaml 
rm install/openshift/99_openshift-cluster-api_worker-machineset-2.yaml
```

We now need to update the worker machineset so that the cluster can leverage the STS tokens to automatically deploy the worker nodes. The following things will need to be changed in the _install/openshift/99_openshift-cluster-api_worker-machineset-0.yaml_ file.

- change spec.replicas to 3
- Update spec.template.spec.providerSpec.value.iamInstanceProfile.id to match worker one made manually with the sg_roles CF template
- double check spec.template.spec.providerSpec.value.instanceType is set to what you want
- double check spec.template.spec.providerSpec.value.placement is set to the proper availabilityZone and region
- Take Note of the value spec.template.spec.providerSpec.value.securityGroups.filters.name.values - this will be used later
- Take Note of the value spec.template.spec.providerSpec.value.subnet.filters.name.values - this will be used later

```shell
vi install/openshift/99_openshift-cluster-api_worker-machineset-0.yaml 
```

We will be using AWS Network Load Balancers created using CloudFormation templates, or manually created through the AWS console. We need to update the cluster ingress configuration files to ensure that OpenShift does not deploy its own AWS load-balancer. Start by editing the `install/manifests/cluster-dns-02-config.yml` and remove all private and public zones.

```shell
vi install/manifests/cluster-dns-02-config.yml
# remove private and public zones if listed
```

With the DNS configuration updated, we will now update our ingress controller files. First edit _install/manifests/cluster-ingress-02-config.yml_ and ensure that the endPointPublishingStrategy is set to HostNetwork.

```shell
vi install/manifests/cluster-ingress-02-config.yml
# spec:
#   domain: apps.cfbuild.example.com
#   endpointPublishingStrategy:
#     type: HostNetwork
```

We will make a similar change to the _install/manifests/cluster-ingress-default-ingresscontroller.yaml_ file.

```shell
vi install/manifests/cluster-ingress-default-ingresscontroller.yaml
# spec:
#   endpointPublishingStrategy:
#     loadBalancer:
#       scope: Internal
#     type: HostNetwork
```

With all our manifest files updated, we will create our ignition files which will be used to configure our bootstrap node.

Create ignition configs

```shell
openshift-install create ignition-configs --dir=install
```

## Add tags to existing Subnet and SecurityGroups

In order for the Machine Controller to create new worker nodes as part of the install, you need to ensure that your AWS Subnets and SecurityGroups are properly labeled for the OpenShift Machine Controller to properly find them. Using the values that you got from the machineset-o.yaml file in the previous steps ensure the following:

| AWS Object    | Tag Name | MachineSet Value                                                         |
| ------------- | --------- | ------------------------------------------------------------------------ |
| SecurityGroup | Name      | spec.template.spec.providerSpec.value.securityGroups.filters.name.values |
| Subnet        | Name      | spec.template.spec.providerSpec.value.subnet.filters.name.values         |

## Stage bootstrap ignition file

Place the bootstrap ignition file somewhere it can be accessed by the bootstrap node. In the instructions below we will use S3:

```shell
$ cd ../install
$ aws s3 mb s3://awssts-infra
$ aws s3 cp install/bootstrap.ign s3://awssts-infra/bootstrap.ign --acl public-read
$ aws s3 ls s3://awssts-infra
```

## Create Network Load Balancer

```
export AWS_DEFAULT_OUTPUT="text"
$ aws cloudformation create-stack --stack-name cfbuildint-nwlb \
     --template-body file://nw_lb.yaml \
     --parameters file://nw_lb.json 
$ aws cloudformation describe-stacks --stack-name cfbuildint-nwlb
```

>**External DNS Steps**
>If you are using an external DNS server you will need to create CNAMES that point to the AWS LB instances that were created
>  * use the "int" LB and point to both api and api-int
>  * use the "ingress" LB and point to *.apps.

## Build Bootstrap node

update bootstrap.json with the updated SecurityGroups, SubnetID, VPC, and bootstrap location
run the following command to create the bootstrap node:

```shell
aws cloudformation create-stack --stack-name awssts-bootstrap \
     --template-body file://cftemplates/bootstrap.yaml \
     --parameters file://cftemplates/bootstrap.json \
     --capabilities CAPABILITY_IAM
```

**Manually UPDATE the "int" loadbalancer to have the newly created bootstrap node added here**
     a. from AWS Console, select EC2->Load Balancers->Target Groups
     b. find your int 6443 target, add the bootstrap node ip address
     c. find your int 22623 target, add the bootstrap node ip address

## Build Control Plane

update control_plane.json with the updated SecurityGroups, SubnetID, VPC, and bootstrap location and Certificate Authority
run the following command to create the bootstrap node:

```shell
aws cloudformation create-stack --stack-name awssts-control-plane \
     --template-body file://cftemplates/control_plane.yaml \
     --parameters file://cftemplates/control-plane.json
```

Once IP addresses have  been assigned to the master nodes, update the int and ext load balancer targetGroups with these IPs

## Monitor the build


```shell
openshift-install wait-for bootstrap-complete --dir=install
INFO Waiting up to 20m0s for the Kubernetes API at https://api.awssts.sandbox39.opentlc.com:6443... 
INFO API v1.22.2+c8538fc up                       
INFO Waiting up to 30m0s for bootstrapping to complete... 
INFO It is now safe to remove the bootstrap resources 
INFO Time elapsed: 6m7s
```

When the bootstrap node is done, you can remove the bootstrap node:

```shell
aws cloudformation delete-stack --stack-name awssts-bootstrap
```

Monitor the system and watch for Worker nodes to be built

Be sure to update your loadbalancer targetGroups and remove the initial bootstrap IP.

## References

[AWS STS](https://docs.openshift.com/container-platform/4.9/authentication/managing_cloud_provider_credentials/cco-mode-sts.html)

[Installing in Restricted AWS Networks](https://docs.openshift.com/container-platform/4.9/installing/installing_aws/installing-restricted-networks-aws.html)

[Installing in Existing AWS VPC](https://docs.openshift.com/container-platform/4.9/installing/installing_aws/installing-aws-vpc.html)

[Installing on AWS with Customizations](https://docs.openshift.com/container-platform/4.9/installing/installing_aws/installing-aws-customizations.html)

https://access.redhat.com/solutions/4713611