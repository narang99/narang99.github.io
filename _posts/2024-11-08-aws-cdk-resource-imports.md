---
layout: post
title: AWS CDK and imported resources
subtitle: Things to look out for when importing resources in AWS CDK
tags: [aws, cloudformation, aws-cdk]
---

[AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html) is an amazing toolkit for managing cloud infrastructure, especially when you do not have any dedicated infrastructure teams. That being said, the abstraction can sometimes have unexpected side effects.  
In this blog, I'm keeping track of some footguns which I've come across when importing resources in CDK. Specifically, **whenever you import a resource in CDK, it can override the associated security group's existing outbound rules, leading to sudden connection failures across your VPC.**    

This happens when:
- You import a resource with a security group that has the default outbound rule (Allow all traffic)
- You try to allow connections from your stack's resources to the imported resource, like below: 
    ```python
    imported_resource.connections.allow_from(managed_resource, port)
    ```

The above action would add an ingress rule in the imported resource's security group from your stack's resource. **It would however, also add an entry in the outbound rules of the imported resource's security group, allowing it to connect to your resources. This would remove the default allow-all outbound rule, resulting in the imported resource losing internet access.**   


The solution is to import the security group associated with the imported resource explicitly and manage its connections. You essentially need to set `allow_all_outbound` and `allow_all_ipv6_outbound` to `True`  
I'm keeping a list of correct ways to import your resources in CDK here (examples are in python)  

## Plain old security-groups

### Bad
This is really the most obvious way of importing, its not safe  
```python
SecurityGroup.from_lookup_by_id(scope, "id", "sg-<id>")
```

### Good
```python
security_group = ec2.SecurityGroup.from_security_group_id(
    scope,
    "id",
    "sg-<id>",
    allow_all_outbound=True,
    allow_all_ipv6_outbound=True,
)

```

## RDS
There is only one way to import RDS (thank god), and all you need to do is correctly import its associated security group as done above  
### Good
```python
security_group = ec2.SecurityGroup.from_security_group_id(
    scope,
    "sg-id",
    "sg-<id>", # the associated sg for the instance you are importing
    allow_all_outbound=True,
    allow_all_ipv6_outbound=True,
)

instance=rds.DatabaseInstance.from_database_instance_attributes(
    scope,
    "rds-id",
    instance_identifier="<instance_identifier>",
    instance_endpoint_address="<instance_endpoint_address>",
    instance_resource_id="<instance_resource_id>",
    port=5432,
    engine=rds.DatabaseInstanceEngine.postgres(
        version=rds.PostgresEngineVersion.VER_16_4, # replace with your version similarly

    ),
    security_groups=[security_group],
)
```

## Application Load Balancer

### Bad
```python
elbv2.ApplicationLoadBalancer.from_lookup(
    scope,
    "id",
    <load-balancer-arn>,
)
```

### Good
```python
elb = elbv2.ApplicationLoadBalancer.from_application_load_balancer_attributes(
    scope,
    "id",
    load_balancer_arn=load_balancer_arn,
    security_group_id="sg-<id>",
    security_group_allows_all_outbound=True,
    vpc=vpc,
)
```

## Application Load Balancers: Listeners

### Bad
```python
elbv2.ApplicationListener.from_lookup(
    scope, "id", listener_arn=listener_arn
)
```

### Good
```python
elbv2.ApplicationListener.from_application_listener_attributes(
    scope,
    id,
    listener_arn=listener_arn,
    security_group=ec2.SecurityGroup.from_security_group_id(
        scope,
        "sg",
        "sg-<id>",
        allow_all_outbound=True,
        allow_all_ipv6_outbound=True,
    ),
)
```
