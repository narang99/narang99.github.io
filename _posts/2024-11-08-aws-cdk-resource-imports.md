---
layout: post
title: AWS CDK and imported resources
subtitle: Things to look out for when importing resources in AWS CDK
tags: [aws, cloudformation, aws-cdk]
---

I'm going to talk about a very specific and annoying problem I've come across when using [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html): **Importing existing AWS resources in your stack can OVERWRITE the outbound/egress rules of active security groups in your live environments**   

This happens when:
- You import an existing resource (say like your RDS, or your ELB) in your stack
- The security group associated with your imported resource has the default outbound rules, where All Traffic is allowed to flow (a very common scenario) and there is no other outbound rule
- You try to allow connections from your stack's resources to the imported resource, like below: 
    ```python
    imported_resource.connections.allow_from(managed_resource, port)
    ```
In this case, CDK would add an outbound rule in the imported resource's security group ***and would remove the 'Allow all traffic' outbound rule***

---
## Some crying
*[Jump [here](#importing-resources-the-safe-way) if you don't like a good tragic story]*  

<img class="floating-right-picture" src="/assets/crying-full-duck.png">
If this hits you, suddenly the impacted environment can get its internet access revoked (AND access to other resources in the same VPC). **You would be lucky if it happens to you in a testing environment, but chances are, your fixes won't be enough, you might not reproduce it again in your test environment, and would suddenly face the same issue in production.** Panic 🥺

This is the timeline of what happened to me:
- I `cdk deploy`ed to my test environment, internet access went away, QA stalled, got some friendly jibes, we fixed it, all good
- I realised it happened because I was importing Security Groups the wrong way, fixed it manually.
  - I did not see the bug again in this environment
  - Since there was no "diff" in my subsequent deployments affecting the problematic resources, I never saw the bug again  
  - This makes it difficult to reproduce the bug, it gives the false impression that your stack is correctly deployable in other places
- I deployed to an existing VPC in production **and it happened again** 😭
- I realised it was because I was importing the Load Balancer wrongly too

We share resources for saving costs (for example, attach a listener using CDK instead of creating a new Load Balancer). If you can get away by creating clean, pure and beautiful stacks using CDK, there is nothing to worry about.  

<img class="floating-right-picture" src="/assets/honking-goose.png">
# Importing resources: the safe way
I'm keeping a list of correct ways to import your resources in CDK here (examples are in python)  
The whole list assumes you want to allow all outbound connections from the associated security group in question  

Key ideas:
- **For all resource imports, import the associated security group with them, and set `allow_all_outbound=True, allow_all_ipv6_outbound=True`**  
- CDK provides ways to do this for every resource I've had an issue with until now, most likely the method would be named something like `from_instance_attributes`, you should prefer to use these methods by default if they exist  
  - using `from_lookup` can almost always trip you



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
Aaaah my nemesis 🥲

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
Similar to ALB, I had this fixed when I saw my fault with ALB

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
