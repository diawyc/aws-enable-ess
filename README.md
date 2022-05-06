# Enable aws ess with organization CLI
CLI command to enable aws ess services within organizations
## Enable guardduty
### Organization management account CLI command:
#### 参数设置:
```
regions=($(aws ec2 describe-regions --query 'Regions[*].RegionName' --output text))
```
或者使用下表删除不需要的regions
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
将admin account id(12位数字)替换下边命令中的999999999999
指定管理员账户:
```
for region in $regions; do
aws guardduty create-detector --data-sources   S3Logs={Enable=true},Kubernetes={AuditLogs={Enable=true}} --enable --finding-publishing-frequency FIFTEEN_MINUTES --region=$region
AWS  guardduty enable-organization-admin-account --admin-account-id 999999999999 --region=$region 
echo $region $(aws guardduty list-organization-admin-accounts --region=$region) $(aws guardduty list-detectors --region=$region --output text --query 'DetectorIds' )
done
```
admin account执行的CLI Command:
参数设置:
Regions需要与上一步management CLI中指定的完全一致
```
regions=($(aws ec2 describe-regions --query 'Regions[*].RegionName' --output text))
```
members.json会在下边CLI command中第一步中生成.

执行CLI命令,将所有成员账号member accounts开启Guardduty所有功能:
```
aws organizations list-accounts  --query 'Accounts[*].{AccountId:Id,Email:Email}' --output json --region=$regions[1]> members.json
for region in $regions; do
AWS  guardduty create-members --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region)  --account-details  file://members.json --region=$region
AWS  guardduty update-organization-configuration --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region)   --auto-enable --data-sources S3Logs={AutoEnable=true},Kubernetes={AuditLogs={AutoEnable=true}} --region=$region
echo $region
aws guardduty update-detector --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region) --data-sources   S3Logs={Enable=true},Kubernetes={AuditLogs={Enable=true}} --enable --finding-publishing-frequency FIFTEEN_MINUTES --region=$region
done
```
