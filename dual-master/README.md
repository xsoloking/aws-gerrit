# Gerrit dual-master in High-Availability

This set of templates provides all the components to deploy a Gerrit dual-master
in HA in ECS. The 2 masters will share the Git repositories via NFS, using EFS.

## Architecture

Four templates are provided in this example:
* `cf-cluster`: define the ECS cluster and the networking stack
* `cf-service-master-1`: define the service stack running Gerrit master 1
* `cf-service-master-2`: define the service stack running Gerrit master 2
* `cf-dns-route`: define the DNS routing for the service

### Networking

* Single VPC:
 * CIDR: 10.0.0.0/16
* Single Availability Zone
* 1 public Subnets:
 * CIDR: 10.0.0.0/24
* 1 public NLB exposing:
 * Gerrit master 1 HTTP on port 8080
 * Gerrit master 1 SSH on port 29418
* 1 public NLB exposing:
 * Gerrit master 2 HTTP on port 8081
 * Gerrit master 2 SSH on port 39418
* 1 Internet Gateway
* 2 type A alias DNS entry, for Gerrit master 1 and 2
* A wildcard SSL certificate available in [AWS Certificate Manager](https://aws.amazon.com/certificate-manager/)

### Data persistency

* EBS volumes for:
  * Indexes
  * Caches
  * Data

### Deployment type

* Latest Gerrit version deployed using the official [Docker image](https://hub.docker.com/r/gerritcodereview/gerrit)
* Application deployed in ECS on a single EC2 instance

### Logging

* All the logs are forwarded to AWS CloudWatch in the LogGroup with the cluster
  stack name

## How to run it

### Setup

The `setup.env.template` is an example of setup file for the creation of the stacks.

Before creating the stacks, create a `setup.env` in the `Makefile` directory and
set the correct values of the environment variables.

This is the list of available parameters:

* `DOCKER_REGISTRY_URI`: Mandatory. URI of the Docker registry. See the
  [prerequisites](#prerequisites) section for more details.
* `SSL_CERTIFICATE_ARN`: Mandatory. ARN of the wildcard SSL Certificate, covering both master nodes.
* `CLUSTER_STACK_NAME`: Optional. Name of the cluster stack. `gerrit-cluster` by default.
* `SERVICE_MASTER1_STACK_NAME`: Optional. Name of the master 1 service stack. `gerrit-service-master-1` by default.
* `SERVICE_MASTER2_STACK_NAME`: Optional. Name of the master 2 service stack. `gerrit-service-master-2` by default.
* `DNS_ROUTING_STACK_NAME`: Optional. Name of the DNS routing stack. `gerrit-dns-routing` by default.
* `HOSTED_ZONE_NAME`: Optional. Name of the hosted zone. `mycompany.com` by default.
* `MASTER1_SUBDOMAIN`: Optional. Name of the master 1 sub domain. `gerrit-master-1-demo` by default.
* `MASTER2_SUBDOMAIN`: Optional. Name of the master 2 sub domain. `gerrit-master-2-demo` by default.
* `CLUSTER_DESIRED_CAPACITY`: Optional.  Number of EC2 instances composing the cluster. `1` by default.

### Prerequisites

The prerequisites to run this stack are:
* a registered and correctly configured domain in
[Route53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/getting-started.html)
* to [publish the Docker image](#publish-custom-gerrit-docker-image) with your
Gerrit configuration in AWS ECR
* to [add Gerrit secrets](#add-gerrit-secrets-in-aws-secret-manager) in AWS Secret
Manager
* an SSL Certificate in AWS Certificate Manager (you can find more information on
  how to create and handle certificates in AWS [here](https://aws.amazon.com/certificate-manager/getting-started/)

### Add Gerrit Secrets in AWS Secret Manager

[AWS Secret Manager](https://aws.amazon.com/secrets-manager/) is a secure way of
storing and managing secrets of any type.

The secrets you will have to add are the Gerrit SSH keys and the Register Email
Private Key set in `secure.config`.

#### SSH Keys

The SSH keys you will need to add are the one usually created and used by Gerrit:
* ssh_host_ecdsa_384_key
* ssh_host_ecdsa_384_key.pub
* ssh_host_ecdsa_521_key
* ssh_host_ecdsa_521_key.pub
* ssh_host_ecdsa_key
* ssh_host_ecdsa_key.pub
* ssh_host_ed25519_key
* ssh_host_ed25519_key.pub
* ssh_host_rsa_key
* ssh_host_rsa_key.pub

Plus a key used by the replication plugin:
* replication_user_id_rsa
* replication_user_id_rsa.pub

You will have to create the keys and place them in a directory.

#### Register Email Private Key

You will need to create a secret and put it in a file called `registerEmailPrivateKey`
in the same directory of the SSH keys.

#### LDAP Password

You will need to put the admin LDAP password in a file called `ldapPassword`
in the same directory of the SSH keys.

#### SMTP Password

You will need to put the SMTP password in a file called `smtpPassword`
in the same directory of the SSH keys.

#### Import into AWS Secret Manager

You can now run the [script](../gerrit/add_secrets_aws_secrets_manager.sh) to
upload them to AWS Secret Manager:
`add_secrets_aws_secrets_manager.sh /path/to/your/keys/directory`

### Publish custom Gerrit Docker image

* Create the repository in the Docker registry:
  `aws ecr create-repository --repository-name aws-gerrit/gerrit`
* Set the Docker registry URI in `DOCKER_REGISTRY_URI`
* Create a `gerrit.setup` and set the correct parameters
 * An example of the possible setting are in `gerrit.setup.template`
 * The structure and parameters of `gerrit.setup` are the same as a normal `gerrit.config`
 * Refer to the [Gerrit Configuration Documentation](https://gerrit-review.googlesource.com/Documentation/config-gerrit.html)
   for the meaning of the parameters
* Add the plugins you want to install in `./gerrit/plugins`
* Publish the image: `make gerrit-publish`

### Getting Started

* Create a key pair to access the EC2 instances in the cluster:

```
aws ec2 create-key-pair --key-name gerrit-cluster-keys \
  --query 'KeyMaterial' --output text > gerrit-cluster.pem
```

*NOTE: the EC2 key pair are useful when you need to connect to the EC2 instances
for troubleshooting purposes. Store them in a `pem` file to use when ssh-ing into your
instances as follow: `ssh -i yourKeyPairs.pem <ec2_instance_ip>`*

* Create the cluster, services and DNS routing stacks:

```
make create-all
```

### Cleaning up

```
make delete-all
```

### Access your Gerrit instances

Get the URL of your Gerrit master instances this way:

```
aws cloudformation describe-stacks \
  --stack-name <SERVICE_MASTER1_STACK_NAME> \
  | grep -A1 '"OutputKey": "CanonicalWebUrl"' \
  | grep OutputValue \
  | cut -d'"' -f 4

aws cloudformation describe-stacks \
  --stack-name <SERVICE_MASTER2_STACK_NAME> \
  | grep -A1 '"OutputKey": "CanonicalWebUrl"' \
  | grep OutputValue \
  | cut -d'"' -f 4
```

Gerrit master instance ports:
* HTTP `8080`
* SSH `29418`

# External services

This is a list of external services that you might need to setup your stack and some suggestions
on how to easily create them.

## SMTP Server

If you need to setup a SMTP service Amazon Simple Email Service can be used.
Details how setup Amazon SES can be found [here](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/send-email-set-up.html).

To correctly setup email notifications Gerrit requires ssl protocol on default port 465 to
be enabled on SMTP Server. It is possible to setup Gerrit to talk to standard SMTP port 25
but by default all EC2 instances are blocking it. To enable port 25 please follow [this](https://aws.amazon.com/premiumsupport/knowledge-center/ec2-port-25-throttle/) link.

## LDAP Server

If you need a testing LDAP server you can find details on how to easily
create one in the [LDAP folder](../ldap/README.md).