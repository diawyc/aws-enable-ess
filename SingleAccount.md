#Single Account Multiple Regions
## 参数设置Set Parameter:
```
regions=($(aws ec2 describe-regions --query 'Regions[*].RegionName' --output text --region=ap-southeast-1))
```
或者使用下表删除不需要的regions or you can use below one and delete the regions you do not want from the list
```
regions=( 
    "us-east-1" 
    "us-east-2" 
    "us-west-1" 
    "us-west-2"
    "eu-north-1" 
    "eu-central-1" 
    "eu-west-3" 
    "eu-west-2" 
    "eu-west-1" 
    "eu-south-1"
    "ap-east-1"
    "ap-south-1" 
    "ap-northeast-1" 
    "ap-northeast-2" 
    "ap-northeast-3" 
    "ap-southeast-1" 
    "ap-southeast-2"
    "ap-southeast-3"
    "me-south-1"
    "ca-central-1"
    "sa-east-1"
    "af-south-1"
   ) 
 ```
 ## 开启服务 Guardduty
 ```
for region in $regions; do
aws guardduty create-detector --data-sources   S3Logs={Enable=true},Kubernetes={AuditLogs={Enable=true}} --enable --finding-publishing-frequency FIFTEEN_MINUTES --region=$region
echo $region 
done
## 开启服务 Inspector
```
for region in $regions; do
echo $region
aws inspector2 enable --resource-types EC2 ECR --region=$region
done

```
