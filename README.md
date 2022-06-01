# Enable aws ess with organization CLI
CLI command to enable aws ess services within organizations
## Enable guardduty,securityhub,inspector,macie 使用CLI命令在AWS Organizations下自动开启各项ESS服务,共分为两步,Step1 指定delegate admin;Step2 开启所有account 服务
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
 ```
adminid='admin account id(12位数字)'
 ```
指定管理员账户 for Guardduty:
```
for region in $regions; do
aws guardduty create-detector --data-sources   S3Logs={Enable=true},Kubernetes={AuditLogs={Enable=true}} --enable --finding-publishing-frequency FIFTEEN_MINUTES --region=$region
AWS  guardduty enable-organization-admin-account --admin-account-id=$adminid --region=$region 
echo $region $(aws guardduty list-organization-admin-accounts --region=$region) $(aws guardduty list-detectors --region=$region --output text --query 'DetectorIds' )
done
```
指定admin account 管理员账户 for securityhub:
```
for region in $regions; do
AWS  securityhub enable-organization-admin-account --admin-account-id=$adminid --region=$region 
aws securityhub enable-security-hub  --enable-default-standards --region=$region
echo $region $(aws securityhub list-organization-admin-accounts --region=$region --query 'AdminAccounts')
done
```

指定admin account 管理员账户 for Inspector:
```
for region in $regions; do
aws inspector2 enable --resource-types EC2 ECR --region=$region
aws inspector2 enable-delegated-admin-account --delegated-admin-account-id=$adminid --region=$region
echo $region
done
```
指定admin account 管理员账户 for Macie:
```
for region in $regions; do
aws macie2 enable-organization-admin-account --region=$region --admin-account-id=$adminid
echo $region
done
```
指定admin account 管理员账户 for Detective:
```
for region in $regions; do
echo $region 
aws detective  enable-organization-admin-account --account-id $adminid --region=$region
done
```
---------------------------------------------------------------------------------------------------------------------------------
### Delegate admin account CLI command:

#### 所有服务的参数设置:
regions需要与第一步management account CLI中指定的完全一致
```
regions=($(aws ec2 describe-regions --query 'Regions[*].RegionName' --output text))
```
### Guardduty
#### 特殊参数设置:
无
#### admin account 执行CLI命令,将所有成员账号member accounts开启Guardduty所有功能:
```
aws organizations list-accounts  --query 'Accounts[*].{AccountId:Id,Email:Email}' --output json --region=$regions[1]> members.json
for region in $regions; do
AWS  guardduty create-members --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region)  --account-details  file://members.json --region=$region
AWS  guardduty update-organization-configuration --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region)   --auto-enable --data-sources S3Logs={AutoEnable=true},Kubernetes={AuditLogs={AutoEnable=true}} --region=$region
echo $region
aws guardduty update-detector --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region) --data-sources   S3Logs={Enable=true},Kubernetes={AuditLogs={Enable=true}} --enable --finding-publishing-frequency FIFTEEN_MINUTES --region=$region
done
```
----------------------------------------------------------------------------------------------------------------------------------------------------------
### Securityhub
#### 特殊参数设置:
 ```
aggregion='<选定的聚合region如:us-west-1>'
 ```
members.json会自动生成
#### admin account 执行CLI命令,将所有成员账号member accounts开启所有功能:
```
aws organizations list-accounts  --query 'Accounts[*].{AccountId:Id,Email:Email}' --output json --region=$regions[1]> members.json
for region in $regions; do
aws securityhub create-members --account-details file://members.json --region=$region
aws securityhub update-organization-configuration --auto-enable --region=$region
echo $region
done
aws securityhub create-finding-aggregator --region=$aggregion  --region-linking-mode=ALL_REGIONS
```
----------------------------------------------------------------------------------------------------------------------------------------------------------
### Inspector
#### 特殊参数设置:
无
#### admin account 执行CLI命令,将所有成员账号member accounts开启所有功能:
```
for region in $regions; do
echo $region
aws inspector2 enable --resource-types EC2 ECR --region=$region
aws inspector2 update-organization-configuration --auto-enable ec2=true,ecr=true  --region=$region 
done
ids=($(aws organizations list-accounts  --query 'Accounts[*].Id' --output text)) 
for region in $regions; do
echo $region
for id in $ids; do
aws inspector2 associate-member --account-id=$id --region=$region
done
done
```
----------------------------------------------------------------------------------------------------------------------------------------------------------
### Macie
#### 特殊参数设置:
 ```
admemail='<admin account email>'
adminid='<admin account ID> 12位数字' (与第一步的要相同)
 ```
将admin account的信息在邀请列表中去除(不去掉也没关系,会报一个错但不影响其它执行)

#### admin account 执行CLI命令,将所有成员账号member accounts开启macie所有功能:
```
orgids=($(aws organizations list-accounts  --query 'Accounts[*].Id' --output text --region=$regions[1]))
accountids=( ${orgids[*]/$adminid} )
orgemails=($(aws organizations list-accounts  --query 'Accounts[*].Email' --output text --region=$regions[1]))
accountemails=(${orgemails[*]/$admemail}) 
len=${#accountids[*]}
for region in $regions; do
for ((i=1; i<=len; i++))
do
aws macie2 create-member --region=$region --account accountId=$accountids[i],email=$accountemails[i]
aws macie2 update-organization-configuration --region=$region --auto-enable
aws macie2  put-findings-publication-configuration --security-hub-configuration publishClassificationFindings=true,publishPolicyFindings=true  --region=$region 
done
echo $region
done
```
-------------------------------------------------------------------------------------------------------------------------------------------------------
### Detective
#### 特殊参数设置:
 ```
admemail='<admin account email>'
adminid='<admin account ID> 12位数字'
 ```
将admin account的信息在邀请列表中去除
#### admin account 执行CLI命令,将所有成员账号member accounts中的detective开启,并允许未来新member自动开启:
```
orgids=($(aws organizations list-accounts  --query 'Accounts[*].Id' --output text --region=$regions[1]))
accountids=( ${orgids[*]/$adminid} )
orgemails=($(aws organizations list-accounts  --query 'Accounts[*].Email' --output text --region=$regions[1]))
accountemails=(${orgemails[*]/$admemail}) 
len=${#accountids[*]}
for region in $regions; do
echo $(aws detective create-graph --region=$region --query "GraphArn" --output text)
echo $region 
for ((i=1; i<=len; i++))
do
aws detective create-members --graph-arn=$(aws detective list-graphs  --region=$region --query "GraphList[0].Arn" --output text) --region=$region --accounts AccountId=$accountids[i],EmailAddress=$accountemails[i]
done
aws detective update-organization-configuration --graph-arn=$(aws detective list-graphs  --region=$region --query "GraphList[0].Arn" --output text) --auto-enable  --region=$region
done   
```
    
