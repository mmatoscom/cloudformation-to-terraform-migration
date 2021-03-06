# AWS cloudformation to terraform migration

This repository shows how to migrate resources managed by AWS cloudformation
to terraform without actually destroying the resources and creating them again.
To accomplish this, AWS cloudformation's `DeletionPolicy: Retain` and `terraform import`
is leveraged.

## 1.
[Show changed files in this step](https://github.com/ulich/cloudformation-to-terraform-migration/blob/step-1/cloudformation-template.json)

First we create a cloudformation stack whose resources we want to migrate to terraform:

```
git checkout step-1
```

Now let's create the inital stack

```
aws cloudformation create-stack --stack-name migrationtest --template-body file://cloudformation-template.json
```


## 2.
[Show changed files in this step](https://github.com/ulich/cloudformation-to-terraform-migration/commit/10ba7265ac7299424c1685e0ef89339d4e04c155#diff-98ff76c6620131d390ffe0c488b996db)

Next we add `"DeletionPolicy" : "Retain",` to the resources we want to migrate:

```
git checkout step-2
```

Here we added it to all resources, because we will migrate all of the resources in one step,
but it could also be done resource by resource.

```
aws cloudformation update-stack --stack-name migrationtest --template-body file://cloudformation-template.json
```


## 3.
[Show changed files in this step](https://github.com/ulich/cloudformation-to-terraform-migration/commit/b8bab6b05c05685ba4a676955e75201278e27be6#diff-0e98d84c23e702e3baa05095730661b0)

Now we create a small terraform project that just defines the resources we want to migrate without any configuration:

```
git checkout step-3
```

We initialize the terraform project:

```
terraform init
```

and we can import the existing resouces into terraform's state. This is done with the command
```
terraform import <type>.<resource name> <aws id>
```

If the resource is defined in a module, it would be
```
terraform import module.<module name>.<resource name> <aws id>
```

The format of the aws ID is always specific to the resource type. For ElasticBeanstalk,
the application's ID is its name, the environment is an auto-generated ID which can be
identified via the aws cli for example.

```
aws elasticbeanstalk describe-environments
``` 

Identify the environment you want to migrate and note the `EnvironmentId`.

In our example it would be:

```
terraform import aws_elastic_beanstalk_application.my_ebs coffee-service
terraform import aws_elastic_beanstalk_environment.my_ebs_staging e-pcpeep3b3z
```

Terraform now imported the resources into it's state.


## 4.
[Show changed files in this step](https://github.com/ulich/cloudformation-to-terraform-migration/commit/9361f081d8f3a501a957b4c5a9ec511115460c94#diff-0e98d84c23e702e3baa05095730661b0)

Now we need to add the properties to the terraform resource descriptions:

```
git checkout step-4
```

The terraform resource descriptions cannot be generated automatically
when importing a resource as of this writing. Instead it must be written manually.

You can now run
```
terraform plan
```

Ideally terraform says that there are no changes required, because the resource
description in the terraform script is the same as the imported resource.

If it does not, you can continue updating the terraform script until `terraform plan`
says that the infrastructure is up to date.

This is the result when running `terraform plan` in this example:

```
  ~ aws_elastic_beanstalk_environment.my_ebs_staging
      wait_for_ready_timeout: "" => "20m"
```

Terraform only wants to update `wait_for_ready_timeout` to 20 minutes since
terraform has a default value of 20 minutes set, even if you do not specify
the value. This plan is safe to be applied, it won't do anything harmful.


## 5.

Now we can remove the resources from the cloudformation stack template,
do another `aws cloudformation update-stack` and the resources will not
be deleted due to the `DeletionPolicy` we added in step 2. 

In our case, we can even delete the whole stack because we migrated all
the resources of the whole stack:

```
aws cloudformation delete-stack --stack-name migrationtest
```
