# Enable aws ess with organization CLI
CLI command to enable aws ess services within organizations
The detail process is from offical document,you may choose to follow the console setting process,or use this CLI commands
https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-accounts.html
## Enable guardduty,securityhub,inspector,macie 使用CLI命令在AWS Organizations下自动开启各项ESS服务,共分为两步,
Step1 set one delegate admin(all service must use the same admin account);
Step2 Enable services in all member accounts and set configuration
### Organization management account CLI command:
#### 参数设置Set Parameter:
```
regions=($(aws ec2 describe-regions --query 'Regions[*].RegionName' --output text --region=us-east-1))
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
 ```
adminid='admin account id(12位数字digital number)'
 ```
All the services should use one same admin account in your organizations except Macie, this service allows you choose a seperate admin account on condition that your DPO team must be seperate with security.
## Guardduty
指定管理员账户Set a delegated admin account for Guardduty:
```
for region in $regions; do
echo $region
aws guardduty create-detector --data-sources   S3Logs={Enable=true},Kubernetes={AuditLogs={Enable=true}} --enable --finding-publishing-frequency FIFTEEN_MINUTES --region=$region
AWS  guardduty enable-organization-admin-account --admin-account-id=$adminid --region=$region 
echo $(aws guardduty list-organization-admin-accounts --region=$region) $(aws guardduty list-detectors --region=$region --output text --query 'DetectorIds' )
done
```
## securityhub
注意：打开之后24小时内一定要在Org内打开config。使用方法为直接通过cloudformation stacksets-sample template中选取
## securityhub
指定admin account 管理员账户Set a delegated admin account for securityhub:
```
for region in $regions; do
echo $region
AWS securityhub enable-organization-admin-account --admin-account-id=$adminid --region=$region 
aws securityhub enable-security-hub  --enable-default-standards --region=$region
echo $(aws securityhub list-organization-admin-accounts --region=$region --query 'AdminAccounts')
done
```
## Inspector
指定admin account 管理员账户 Set a delegated admin account for Inspector:
```
for region in $regions; do
echo $region
aws inspector2 enable --resource-types EC2 ECR --region=$region
aws inspector2 enable-delegated-admin-account --delegated-admin-account-id=$adminid --region=$region
echo $(aws inspector list-organization-admin-accounts --region=$region --query 'AdminAccounts')
done
```
## Macie
指定admin account 管理员账户 Set a delegated admin account for Macie:
```
for region in $regions; do
echo $region
aws macie2 enable-organization-admin-account --region=$region --admin-account-id=$adminid
echo $(aws macie list-organization-admin-accounts --region=$region --query 'AdminAccounts')
done
```
## Detective
指定admin account 管理员账户 Set a delegated admin account for Detective:
```
for region in $regions; do
echo $region 
aws detective  enable-organization-admin-account --account-id $adminid --region=$region
echo $(aws detective list-organization-admin-accounts --region=$region --query 'AdminAccounts')
done
```
---------------------------------------------------------------------------------------------------------------------------------
### Delegate admin account CLI command:

#### 所有服务的参数设置 Set parameter for all services:
regions需要与第一步management account CLI中指定的完全一致 must be the same as the list in step1
```
regions=($(aws ec2 describe-regions --query 'Regions[*].RegionName' --output text --region=us-east-1))
```
### Guardduty
#### 特殊参数设置Set special paramter for certain service:
无
#### admin account run below command to enable the service in all accounts执行CLI命令,将所有成员账号member accounts开启Guardduty所有功能:
Type1 这种会报错（管理员账号不能加入自己）但不影响结果
```
aws organizations list-accounts  --query 'Accounts[*].{AccountId:Id,Email:Email}' --output json --region=$regions[1]> members.json
for region in $regions; do
AWS  guardduty create-members --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region)  --account-details  file://members.json --region=$region
AWS  guardduty update-organization-configuration --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region)   --auto-enable --data-sources S3Logs={AutoEnable=true},Kubernetes={AuditLogs={AutoEnable=true}} --region=$region
echo $region
aws guardduty update-detector --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region) --data-sources   S3Logs={Enable=true},Kubernetes={AuditLogs={Enable=true}} --enable --finding-publishing-frequency FIFTEEN_MINUTES --region=$region
done
```
Type 2 这种不会有报错
```
orgids=($(aws organizations list-accounts  --query 'Accounts[*].Id' --output text ))
accountids=( ${orgids[*]/$adminid} )
orgemails=($(aws organizations list-accounts  --query 'Accounts[*].Email' --output text ))
accountemails=(${orgemails[*]/$admemail})
len=${#accountids[*]}
echo $len
```
```
for region in $regions; do
echo $region
for ((i=1; i<=len; i++));do
echo $accountids[i] $accountemails[i]
aws guardduty create-members --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region)  --account-details AccountId=$accountids[i],Email=$accountemails[i]  --region=$region
aws guardduty update-detector --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region) --data-sources   S3Logs={Enable=true} --enable --finding-publishing-frequency FIFTEEN_MINUTES --region=$region
aws guardduty update-organization-configuration --detector-id $(aws guardduty list-detectors --output text --query 'DetectorIds' --region=$region)   --auto-enable --data-sources S3Logs={AutoEnable=true} --region=$region
done
done
```

----------------------------------------------------------------------------------------------------------------------------------------------------------
### Securityhub
#### Set special paramter for certain service特殊参数设置:
 ```
aggregion='<选定的聚合aggregated region如:us-west-1>'
adminid=$(aws sts get-caller-identity --output text --query 'Account' )
 ```
members.json会自动生成
#### admin account run below command to enable the service in all account 执行CLI命令,将所有成员账号member accounts开启所有功能:
Type1 这种会报错（管理员账号不能加入自己）但不影响结果
```
aws organizations list-accounts  --query 'Accounts[*].{AccountId:Id,Email:Email}' --output json --region=$regions[1]> members.json
for region in $regions; do
echo $region
aws securityhub create-members --account-details file://members.json --region=$region
aws securityhub enable-security-hub  --enable-default-standards --region=$region
aws securityhub update-organization-configuration --auto-enable --region=$region

done
aws securityhub create-finding-aggregator --region=$aggregion  --region-linking-mode=ALL_REGIONS
```
Type 2 这种不会有报错
```
orgids=($(aws organizations list-accounts  --query 'Accounts[*].Id' --output text --region=$regions[1]))
accountids=( ${orgids[*]/$adminid} )
len=${#accountids[*]}
for region in $regions; do
echo $region
for ((i=1; i<=len; i++));do
echo $accountids[i]
aws securityhub create-members --account-details AccountId=$accountids[i] --region=$region
aws securityhub enable-security-hub  --enable-default-standards --region=$region
aws securityhub update-organization-configuration --auto-enable --region=$region
done
done
aws securityhub create-finding-aggregator --region=$aggregion  --region-linking-mode=ALL_REGIONS
```
----------------------------------------------------------------------------------------------------------------------------------------------------------
### Inspector
#### 特殊参数设置Set special parameter for this service:
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
#### 特殊参数设置Set special parameter for this service:
 ```
admemail='<admin account email>'
adminid='<admin account ID> 12位数字' (与第一步的要相同)
 ```
将admin account的信息在邀请列表中去除(不去掉也没关系,会报一个错但不影响其它执行)

#### admin accountadmin account run below command to enable the service in all account 执行CLI命令,将所有成员账号member accounts开启macie所有功能:
```
orgids=($(aws organizations list-accounts  --query 'Accounts[*].Id' --output text --region=$regions[1]))
accountids=( ${orgids[*]/$adminid} )
orgemails=($(aws organizations list-accounts  --query 'Accounts[*].Email' --output text --region=$regions[1]))
accountemails=(${orgemails[*]/$admemail}) 
len=${#accountids[*]}
for region in $regions; do
echo $region
for ((i=1; i<=len; i++))
do
aws macie2 create-member --region=$region --account accountId=$accountids[i],email=$accountemails[i]
aws macie2 update-organization-configuration --region=$region --auto-enable
aws macie2  put-findings-publication-configuration --security-hub-configuration publishClassificationFindings=true,publishPolicyFindings=true  --region=$region 
done

done
```
-------------------------------------------------------------------------------------------------------------------------------------------------------
### Detective
#### 特殊参数设置Set special parameter for this service:
 ```
admemail='<admin account email>'
adminid='<admin account ID> 12位数字'
 ```
将admin account的信息在邀请列表中去除 remove the admin account in the accountids
#### admin account admin account run below command to enable the service in all account执行CLI命令,将所有成员账号member accounts中的detective开启,并允许未来新member自动开启:
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
    
