---
tags:
  - aws
  - awscli
  - ec2
  - operation
---
aws ec2 实例的操作:
```shell
# list all instances
aws ec2 describe-instances --output table
aws ec2 describe-instances --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name,PrivateIP:PrivateIpAddress,PublicIp:PublicIpAddress,Name: Tags.Name, Zone: Placement.AvailabilityZone}' --output table


```


```shell
#

```





