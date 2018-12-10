# AWS CloudFormation Template Generator

[toc]

## TL;DR

This repository consists of:

* A _Ansible_ playbook
* A number of _Ansible_ templates that generate AWS CloudFormation templates with an external configuration
  file as driver

In combination with the configuration file, the _Ansible_  playbook creates a set of
AWS CloudFormation templates, and deploys these templates to you AWS account.

Example execution command:

```
ansible-playbook CreateOrUpdateEnv.yml --extra-vars 
```

## Dependencies and Prerequisites

* A _Docker_ engine when using the `dockerwrapper` to build and deploy the templates
* A local _Ansible_ and _AWS CLI_ client installation when **not** using the
  `dockerwrapper`
* The templates use resources created by the `VPC.yml` template in
  [this repository](https://github.com/rik2803/aws-cfn-templates). This should disappear
  in the future, but until then, we've got to deal with it.

## Running the playbook in a controlled way

### Background

The Ansible templates used for the creation of the AWS CloudFormation templates, much like
any other kind of code, evolves. Sometimes, evolution has a price, and that price is
backward compatibility.

To make sure that a _aws-cfn-gen_ configuration file will still build after a backward
compatibility breaking change, the build environment should not change in time.

This is solved by using a _Docker_ image to create the AWS CloudFormation templates and
to deploy these AWS CloudFormation templates to the desired account.

The _Docker_ image contains a combination of _ansible_ and _AWS CLI_ versions, and running
the _Docker_ image with the right set of environment variables allows the user to choose
the _tag_ in this repository to checkout for the build and deploy.

The _Docker_ image is called `ixor/ansible-aws-cfn-gen` and can be
found [here](https://hub.docker.com/r/ixor/ansible-aws-cfn-gen/). The documentation for
this _DockerWrapper_ is in rhe file `README_DOCKERWRAPPER.md` in this repository.

### An example _dockerwrapper_ script

If, for example, you have a configuration file, `config.yml` that is tested and approved
with:

  * Ansible v2.6.1
  * AWS CommandLine version 1.6
  * v0.1.0 of the template files in this repository.

Your `dockerwrapper` script will look like this:

```bash
#! /bin/bash

GITTAG=v0.1.0

docker run --rm \
    -e AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID} \
    -e AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}" \
    -e AWS_REGION=${AWS_REGION:-eu-central-1} \
    -e AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN} \
    -e GITTAG=${GITTAG:-v0.1.0} \
    -e ANSIBLE_TAGS=${ANSIBLE_TAGS:-} \
    -e CONFIG=config.yml \
    -v ${PWD}:/config \
    -it \
    ixor/ansible-aws-cfn-gen:2.6.1-aws-1.6
```

To create the _AWS CloudFormation_ template files and deploy them to the AWS account of your choice,
follow these steps:

* Start your (_Docker_) engines
* **Set _AWS Credentials_ for that account**. The roles and policies linked to the user that owns these
  credentials should be able to create all the configured resources in that account. The first step when
  the container is started is to check the account id you are logged in to with the account id in the
  configuration file.  If they do not match, the build is cancelled.
* (**optional**) Set the **environment variables** `ANSIBLE_TAGS` and `ANSIBLE_SKIPTAGS` to limit the
  execution of the playbook to just the services you include minus the service you exclude.
* Run the `dockerwrapper` script created above, by running `bash dockerwrapper`

## Resources that can be created using this repository

* AWS Application Load Balancers (ALB)
* AWS ECR
* AWC ECS Cluster
* AWS ECS Tasks and Services
* S3 buckets
* Route 53 private Hosted Zones
* IAM Users, Roles and Policies
* CloudFront distribution
* DynamoDB tables

## Order of installation is important

The template for certain resources may depend on resources created by other
templates. Those templates obviously have to be deployed for the dependencies
to exist.

Therefor, the order of resource creation in the main playbook (`CreateOrUpdateEnv.yml`)
should not be changed, unless there is a dependency issue and changing the order does
not introduce new dependencies.

## Configuration file

### The project configuration

The project configuation is used throughout the templates, mostly to prefix resource
names and resource logical names to guarantee uniqueness.

All `cfn_name` properties are used to create the CloudFormation logical resource names.
Those name can only include letters and numbers.

```yaml
organization:
  name: Acme
  cfn_name: Acme
```

```yaml
project:
  name: my_big_project-prd
  shortname: mbp-prd
```

### Configuration to _tag_ resources created by _CloudFormation_

These tags can be used in _Cost Explorer_ to create reports per environment and per application, for example.

```yaml
application: mybigapplication
env: prd
```

### The `referenced_stacks`

These are stacks (usually only the VPC stack) created by [these templates](https://github.com/rik2803/aws-cfn-templates),
and used troughout `aws-cfn-gen`. As said in the _Dependencies and Prerequisites_ section, this is meant
to disappear over time.

```yaml
referenced_stacks:
  VPCStackName: "VPCForAcmePrd"
``` 

### Describe the account where de deployment will be done

This is mostly used for sanity checks (are you running on the account you think you are running on?)
and for the region.

```yaml
target_account:
  name: "acme.mybigapplication-prd"
  account_id: "123456789012"
  region: "eu-central-1"
```

### Create _Lambda_ functions

Let's start with an example:

```yaml
  - name: aws-lambda-s3-logs-to-cloudwatch
    handler: handler
    runtime: nodejs8.10
    role: S3ToCloudwatchLogsRole
    code:
      s3_bucket: "{{ lambda_function_bucket_name }}"
      s3_key: aws-lambda-s3-logs-to-cloudwatch-06b0c5cda86555d95f5939bedeca17830c81ff98.zip
    environment:
      - name: LOGGROUP_NAME
        value: lb_access_logs
      - name: LOGSTREAM_NAME
        value: albint
    invoke_permissions:
      - type: predefined
        description: "Allows bucket events to trigger this lambda function"
        name: s3
        bucket_arn: "arn:aws:s3:::ixordocs-dev-accesslogs-albint"
```

**How to use the same function more than once in an environment?**

Sometimes, the same function needs to be used more than once, for example if there are different
triggers or a different set onf environment variables that influence the execution and the result
of the function.

To achieve this, create identical blocks (with different envvars or whatever changes), and the
`name` should have a suffix that starts with un underscore.

#### `name`

The name determines:

* The _CloudFormation_ resource name
* The name of the function (i.e. the name of the file in the zip defined by 
  `'s3://' + code.s3_bucket + '/' + `code.s3_key`)
  
The `name` can contain:

* letters
* numbers
* hyphens
* 0 or 1 underscores, used to differentiate the _CFN_ resource name in case of
  multiple instances of the same function.
  
If the name contains an underscore, the part before the underscore is used to determine
the function name, and the complete string is used, after some _CFN_ related transformation,
as the _CloudFormation_ resource name. 

#### `handler`

#### `runtime`

#### `role`

#### `code.s3_bucket`

#### `code.s3_prefix`

#### `environment`

##### `environment[n].name`

##### `environment[n].value`


### Creation of IAM related resources

Some IAM resources are implicitely created by other components, i.e. in _CloudFront_ to allow
a user to invalidate the _CloudFront_ distributions.

But it is sometimes useful to be able to create your own roles and policies. This can be
accomplished by using these configuration sections:

* `managed_policies`
* `awsroles` (this used to be called `roles`, but _Ansible_ complained about me using its reserved word)
* `iam_users`: Create users, assign policies and (optionally) create credentials

#### `managed_policies`

A list of policies, syntax is identical to the *CloudFormation* syntax for [`AWS::IAM::ManagedPolicy`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-managedpolicy.html).

```yaml
managed_policies:
  - name: S3DeployArtifactsAccess
    policy_document:
      Version: '2012-10-17'
      Statement:
        - Sid: AllowReadAccessToS3CentralDeployArtifacts
          Effect: Allow
          Action:
            - s3:GetObject
            - s3:GetObjectAcl
            - s3:ListBucket
          Resource:
            - arn:aws:s3:::acme-s3-deploy-artifacts
            - arn:aws:s3:::acme-s3-deploy-artifacts/*
```

#### `awsroles`

A list of roles, syntax is identical to the *CloudFormation* syntax for [`AWS::IAM::Role`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html).

```yaml
awsroles:
  - name: MyECSGeneralTaskRole
    policy_arns:
      # The last part in arn:aws:iam::account_id:policy/policyname
      - S3DeployArtifactsAccess
    assumerole_policy_document:
      Version: '2012-10-17'
      Statement:
        - Sid: ''
          Effect: Allow
          Principal:
            Service: ecs-tasks.amazonaws.com
          Action: sts:AssumeRole
```

#### `iam_users`

```yaml
iam_users:
  - name: s3-deploy
    cfn_name: S3DeployUser
    managed_policies:
      - S3DeployArtifactsAccess
    create_accesskeys: true
```

### `ecs`: _Elastic Container Services_

Create an (empty) ECS cluster.

```yaml
ecs:
  cluster:
    keypair: "id_rsa_ixor.ixordocs-prd"
    instance_type: "t2.xlarge"
    cluster_size:
      min: 2
      max: 5
      desired: 2
    ebs_size: 40
    dnm_basesize: 20G
```

#### `ecs.ebs_size`

Override the default 20GB ECS Cluster instance EBS size. Unit is `GB`.

**Downtime Warning**: Changing this setting will cause the launch configuration
to change and will consequently spawn new ECS instances.

#### `ecs.dm_basesize`

Override the default 10GB of thin provisioned container storage. Unit is required (i.e. `20G`)

**Downtime Warning**: Changing this setting will cause the launch configuration
to change and will consequently spawn new ECS instances.

### `loadbalancers`: Create _Application Load Balancers_

`loadbalancers` is a list of, you guessed it, loadbalancers.

It creates a typical loadbalancer, with these components:

* An ALB or *Application LoadBalancer* (`AWS::ElasticLoadBalancingV2::LoadBalancer`).
  This can be an internet-facing loadbalancer (`scheme: internet-facing`), or an internal
  loadbalancer (`scheme: internal`).
* The _Security Groups_ and subnets used for the loadbalancer are extracted from the
  VPC stack mentioned before. That stack uses AWSs reference architecture and matches most setups.
* A HTTP listener on both internet-facing and internal loadbalancers.
* A HTTPS listener on the internet-facing loadbalancer. This requires a certificate for TLS
  termination.
* A default target group for HTTP and HTTPS
* Additional rules and target groups are created for the services defined in
  `applicationconfig`
* (optional) Define redirects, see below in `redirects`

```yaml
loadbalancers:
  - name: ALBExt
    scheme: "internet-facing"
    certificate_arn: "arn:aws:acm:eu-central-1:632928949881:certificate/2e225841-3a4a-41cd-a677-325f7d2cf262"
    def_tg_http_healthcheckpath: /health
    def_tg_https_healthcheckpath: /health
  - name: ALBInt
    scheme: "internal"
    idle_timeout_seconds: 120
    accesslogs:
      state: enabled
      log_expiry_days: 14
      s3_objectcreated_lambda_import: IxordocsDevLambda-AwsLambdaS3LogsToCloudwatchAlbext
```

#### `access_logs`

When `access_logs` is defined and `state` is `enabled`,
following resources are created:

* A S3 bucket named `{{application }}-{{ env }}-accesslogs-{{ lbname }}`
* An lifecycle rule that expires the access logs after `log_expiry_days` days
* A bucket policy that allows the AWS ALB account in the current region to
  write to that bucket
* A `s3.ObjectCreated` trigger to a lambda function if
  `accesslogs.s3_objectcreated_lambda_import` is defined. That *Lambda* function can, for example,
  be used to ship the S3 logs to *CloudWatch*.

And the loadbalancer will get the attributes required to enable access logs, as specified
[here](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_LoadBalancerAttribute.html).

#### `idle_timeout_seconds`

Default is `60`, sets the LB `LoadBalancerAttribute` named `idle_timeout.timeout_seconds` to this
value.

#### `redirects`

```yaml
loadbalancers:
  - name: ALBExtRedirectTest
    ...
    redirects:
      - host_header: "acme.com"
        path-pattern: "/"
        priority: 200
        status_code: HTTP_302
        path: /go_here
        skiproute53: true
```

`redirects` is a list of dicts that define URLs to redirect.

##### `redirects[n].host_header` (required)

A hostname that will trigger the redirect.

##### `redirects[n].path_pattern` (optional)

A `path-pattern` strings that will trigger the redirect.

##### `redirects[n].priority` (required)

Determines the order of the redirect rules.

##### `redirects[n].status_code` (optional, default is `HTTP_301`)

On of these strings:

* `HTTP_301` for permanent redirect
* `HTTP_302` for temporary redirect

##### `redirects[n].to` (optional, default is original host)

##### `redirects[n].path`  (optional, default is original path)

##### `redirects[n].skiproute53`

Skip the creation of a *Route 53* record.


### `route53`

TODO

### `s3`

Create S3 buckets.

Other S3 buckets might be created implicitely by the other components (i.e. _CloudFront_),
but `s3` can be used to explicitely create buckets.


```yaml
s3:
  - name: mybucket
    cfn_name: MyBucket
    access_control: Private
    static_website_hosting: no
    versioning: {Enabled|Suspended}
    lifecycle_configuration: |
      Rules:
        - ExpirationInDays: 14
```

#### `name`

The name for the bucket. The resulting name will be the value of this variable,
prefixed with `{{ application }}-{{ env }}-`.

```yaml
application: mybigapplication
env: prd

...

s3:
  - name: mybucket
    cfn_name: MyBucket
    access_control: Private
    static_website_hosting: no
```

For the above configuration, the resulting bucket will be named `mybigapplication-prd-mybucket`.

#### `cfn_name`

The name to be used for the _CloudFormation_ logical resource.

The final _CloudFormation_ logical name will be `{{ cfn_project }}{{ bucket.cfn_name }}` where
`{{ cfn_project }}`.

#### `access_control`

This setting grants predefined permissions to the bucket. All object created after this setting
was set or updated will get that ACL.

See [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html#canned-acl) for valid values.


#### `static_website_hosting`

Valid values:

* `yes` or `Yes`
* `true` or `True`
* `on` or `On`

All other values will not enable website hosting on the bucket.

**Important**: This potentially exposes object to the evil internet.

#### `versioning`

Enable or disable (suspend) bucket versioning.

Allowed values:

* `Enabled`
* `Suspended`


#### `lifecycle_configuration`

Use the exact same _yaml_ as described in [Amazon S3 Bucket Rule](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket-lifecycleconfig-rule.html).

If `lifecycle_configuration` is not specified, the default lifecycle rule is:

```yaml
        Rules:
          - NoncurrentVersionExpirationInDays: 60
            Status: Enabled
```

### `applicationconfig`

`applicationconfig` is a list of applications to run in the ECS cluster. Each
element in the `applicationconfig` list contains the application description.

```javascript
  - name: "servicename"
    cfn_name: ServiceName
    target: "ecs"
    environment:
      - name: JAVA_OPTS
        value: "-Xmx2048m"
    ecs:
      image: "123456789012.dkr.ecr.eu-central-1.amazonaws.com/example/service:latest"
      containerport: 8080
      memory: 2048
      cpu: 512
      desiredcount: 2
      healthcheckgraceperiodseconds: 3600
      task_role_arn: "arn:aws:iam::123456789012:role/ECSTaskRole"
    lb:
      name: ALBExt
      ### Can be public or private, determines if DNS entries are created in the public
      ### or private hosted zones.
      type: public
      healthcheckpath: /actuator/health
      healthcheckokcode: "200"

    domains:
      - name: example.com
        cfn_name: ExampleCom
        listener_rule_host_header: service.example.com
        priority: 210
      - name: voorbeeld.be
        cfn_name: VoorbeeldBe
        listener_rule_host_header: service.voorbeeld.be
        priority: 211
        skiproute53: false
```

#### `applicationconfig[n].name`

**Important**: The `name` should only contain letters, numbers, hyphens and colons. Underscores are not allowed.

**Important**: The `name` should not be changed one the service was created. If it **is** changed, the service
and the related reources might be recreated and will cause downtime.

The name defines the name to be used for the service. It is alse used to create related resources:

* _Listener Rule_ name in the `ALB.yml` template
* _Target Group_ CloudFormation export for use in other templates in `ALB.yml`
* Name of the _CloudWatch Log Group_ in `ECS.yml`
* Name of the _Task Definition_ in `ECS.yml`

#### `applicationconfig[n].cfn_name`

CloudFormation logical names are restricted to letters and numbers only. All `cfn_` properties are used
for naming _CloudFormation_ resource logical names.

#### `applicationconfig[n].target`

Where the service will be running, `ecs` is currently the only target. For future extensions.

#### `applicationconfig[n].environment`

A list of key-value pairs to add to the environment variables of the running container

```javascript
    environment:
      - name: JAVA_OPTS
        value: "-Xmx2048m"
```

#### `application[n].ecs`

##### `application[n].ecs.image`

The image to run in the container. This can be a ECR repository, or a (public) _Docker Hub_
repository.

Private _Docker Hub_ repositories are not supported at the moment.

##### `application[n].ecs.containerport`

The port the service inside the container is listening on. When the task is started, a port
mapping will be created by the ECS Agent (which also runs in a Docker container), and that
port will be registered with the Target Group to which the service is linked, in order for
loadbalancing to do its job.

##### `application[n].ecs.memory`

The number on MB to reserve for the container. If the container requires more memory than
is available (i.e. not reserved) on any of the ECS cluster nodes, the task will not be started.
This will be logged in the service’s events in de AWS Console.

##### `application[n].ecs.cpu`

The number of CPU shares to allocate to the running container. Each vCPU on AWS
accounts for 1024 CPU shares. The available number of CPU shares in the cluster is
`1024 * sum_of_vCPUs_of_all_clusternodes`.

For a list of vCPUs per instance type, look [here](http://aws.amazon.com/ec2/instance-types/).

##### `application[n].ecs.desiredcount`

The number of instances to start and maintain for that service.

##### `application[n].ecs.deploymentconfiguration`

Describes how the services will behave when a service is redeployed.

##### `application[n].ecs.deploymentconfiguration.max_percent`

The maximum number of tasks, specified as a percentage of the Amazon ECS service's
_DesiredCount_ value, that can run in a service during a deployment. To calculate
the maximum number of tasks, Amazon ECS uses this formula: the value of
`DesiredCount * (the value of the MaximumPercent/100)`, rounded down to the nearest
integer value.

(From the [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ecs-service-deploymentconfiguration.html))

##### `application[n].ecs.deploymentconfiguration.min_healthy_percent`

The minimum number of tasks, specified as a percentage of the Amazon ECS service's
DesiredCount value, that must continue to run and remain healthy during a deployment.
To calculate the minimum number of tasks, Amazon ECS uses this formula: the value of
`DesiredCount * (the value of the MinimumHealthyPercent/100)`, rounded up to the 
nearest integer value.

(From the [AWS Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ecs-service-deploymentconfiguration.html))

#### `application[n].lb`

**Important**: Changing the LB requires the `AWS::ECS::Service` resource to be
recreated. Since the framework assigns a nam to the `AWS::ECS::Service` resource,
this means that the template will fail unless (or):

* the `application[n]` with the changed loadbalancer is first deleted (remove it
  from the configuration file and deploy) and recreated
* **or** the `application[n].name` is changed

##### `application[n].lb.name`

The name of the LoadBalancer behind which the service should be put. External services
should get their traffic from the external load balancer, while internal services should
be put behind an internal load balancer. Internal traffic can be HTTP, while external
traffic should be HTTPS.

##### `application[n].lb.type`

Only used if DNS records have to be made for external services. This only works if the
domain is hosted in _Route 53_, and if the value of the property is `public`.

##### `application[n].lb.healthcheckpath`

The path to check the health of the service.

##### `application[n].lb.healthcheckokcode`

The HTTP code that reflects a healthy services. For example:

* `200`
* `200-299`
* `200-499`
* Values in the `500` range cannot be used


##### `application[n].lb.targetgroup`

### `ecr`: _Elastic Container Registry_

A list of repositories to create on the AWS account. This implicitely creates 2 users,
`ECRPush` and `ECRPull` and their obvious permissions on all those repositories.

```yaml
ecr:
  repositories:
    - name: acme/mydockerimage
      cfn_name: AcmeMyDockerImage
```

### `cloudfront_distributions`

**NOTE** - The certificate must be created in the `us-east-1` region.

* `name`
* `cfn_name`
* `cnames`

```yaml
cloudfront_distributions:
  - name: apps
    cfn_name: Apps
    cnames:
      - "apps.acme.com"
    certificate_arn: "arn:aws:acm:us-east-1:123456789012:certificate/1d2e5c3-2f5e-a8e8dw8-f2c5-5d42170bbe0"
    logging:
      prefix: apps
    origins_and_cachebehaviors:
      - origin_name: "apps-{{ application }}-{{ env }}"
        forward_headers:
          - Origin
        path_pattern: "/*"
        allowed_http_methods: options
        priority: 999
```

#### Working with custom origins

The default behaviour, when no `domain` property is defined, is to use a
S3 bucket as the origin for the distribution.

In that case, the bucket will be implicitely created and be given the name `origin.name`.

When using the `domain` property and `type` `s3`, the bucket is assumed to already exist in
the AWS account where the CloudFormation template is bing deployed.


```yaml
    origins_and_cachebehaviors:
      - ...
      - origin_name: "{{ application }}-{{ env }}-name"
        domain:
          type: import
          name: TheNameACloudformationExport
          origin_path: /mypath
        ...  
```

```yaml
    origins_and_cachebehaviors:
      - ...
      - origin_name: "{{ application }}-{{ env }}-name"
        domain:
          type: s3
          name: my-bucket
          origin_path: /prefix
        ...  
```

```yaml
    origins_and_cachebehaviors:
      - ...
      - origin_name: "{{ application }}-{{ env }}-name"
        domain:
          type: custom
          origin_domain_name: "google.be"
          orinig_path: "/nl"
        ...  
```

##### `origins_and_cachebehaviors[n].origin_name`

The value of this propery is used to:

* Name the origin
* Implicitely create a bucket with the same name (mind the global uniqueness!!)

##### `origins_and_cachebehaviors[n].domain.type`

Can be one of these values:

* `import`: When `type` is `import`, following properties are allowed:
  * `name`: The name of the _CloudFormation_ export to be used for the import
  * `origin_path` (**optional**): If you want _CloudFront_ to request your content from a 
                                  directory in your Amazon S3 bucket or your custom origin,
                                  enter the directory name here, beginning with a `/`.
                                  _CloudFront_ appends the directory name to the value of
                                  `origin_domain_name` when forwarding the request to your origin,
                                  for example, `myawsbucket/production`. Do not include a `/`
                                  at the end of the directory name. 

* `s3`: Use the value of the `name` property as the name of the bucket in the
        same region as the region where the _CloudFormation_
        stack is being deployed. The domain that will be used by the _CloudFront_
        distribution will be
        `{{ origin.domain.name }}.s3-website.{{ target_account.region }}.amazonaws.com`
  * `name`
  * `origin_path` (**optional**): If you want _CloudFront_ to request your content from a 
                                  directory in your Amazon S3 bucket or your custom origin,
                                  enter the directory name here, beginning with a `/`.
                                  _CloudFront_ appends the directory name to the value of
                                  `origin_domain_name` when forwarding the request to your origin,
                                  for example, `myawsbucket/production`. Do not include a `/`
                                  at the end of the directory name.

* `custom`: When `type` is `custom`, following porperties are allowed:
  * `origin_domain_name`: The name of the domain to use as the origin, for example `google.com`.
  * `origin_path` (**optional**): If you want _CloudFront_ to request your content from a 
                                  directory in your Amazon S3 bucket or your custom origin,
                                  enter the directory name here, beginning with a `/`.
                                  _CloudFront_ appends the directory name to the value of
                                  `origin_domain_name` when forwarding the request to your origin,
                                  for example, `myawsbucket/production`. Do not include a `/`
                                  at the end of the directory name. 

#### Custom Error Responses

```yaml
cloudfront_distributions:
  - name: servicedesk
    ...
    custom_error_responses:
      - error_caching_min_ttl: 300
        error_code: 404
        response_code: 200
        response_page_path: /index.html
```

### `dynamodb`

An example:

```yaml
dynamodb:
  - table_name: journal
    backup: true
    attributes:
      - attribute_name: par
        attribute_type: S
      - attribute_name: num
        attribute_type: N
    key_schema:
      - attribute_name: par
        key_type: HASH
      - attribute_name: num
        key_type: RANGE
    billing_mode: PROVISIONED | PAY_PER_REQUEST
    provisioned_throughput:
      read_capacity_units: 5
      write_capacity_units: 5
```

#### `backup`

This setting enables or disables the PITR for the table.

Allowed values:

* `true`
* `false` (default)

#### `billing_mode` and `provisioned_throughput`

`billing_mode` can have 2 values:

* `PROVISIONED` (default)
* `PAY_PER_REQUEST`

`provisioned_throughput` is ignored if `billing_mode` is `PAY_PER_REQUEST`.

See the [AWS documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dynamodb-table.html#cfn-dynamodb-table-billingmode) for more details.

## Links
