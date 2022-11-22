# AWS ElasticCache
### Overview
This CloudFormation template will setup High availability (MultiAZ) ElasticCache Redis cluster, scheduled automatic backup, special user and user group.
Two nodes to be used as the primary and replica, respectively.
AWS Secret Manager used to store credentials for ElasticCache users and will be retrieved latter by pod running on EKS cluster.
AWS Parameter Store used to store ElasticCache endpoints and will be retrieved latter by pod running on EKS cluster.
Aws KMS used for disk encryption.
ElastiCache subnet groups used the same private subnet groups as the EKS cluster.
## Environment
In this introduction, we will create ElastiCache for Redis (cluster disabled).

## Pre requirements
- AWS account
- EKS cluster
    - ExternalSecrets: repository: https://github.com/external-secrets/external-secrets


## Explanation of key points of this template file
### ElastiCache for Redis (Cluster Disabled)
- Default parameters
```yaml
  CacheEngine:
    Type: String
    Default: 'redis'

  CacheEngineVersion:
    Type: String
    Default: '7.0'

  RedisPort:
    Type: Number
    Default: '6379'
```
- Subnet Group
Specify two or more subnets in the SubnetIds property.
Note that if ElastiCache is multi-AZ, the AZs of the subnets specified here must be separated.
```yaml
  CacheSubnetGroup:
    Type: AWS::ElastiCache::SubnetGroup
    DependsOn: ElastiCacheLogGroup
    Properties:
      CacheSubnetGroupName: !Sub '${AWS::StackName}-subnetgroup'
      Description: !Sub '${AWS::StackName}-ElastiCacheSubnetGroup'
      SubnetIds:
        - !Ref 'SubnetIda'
        - !Ref 'SubnetIdb'
        - !Ref 'SubnetIdc'
```
- Replication Group or Cache Cluster
When creating ElastiCache itself with CloudFormation, you must create either of the following resources
```
Replication Group: AWS::ElastiCache::ReplicationGroup
Cache Cluster: AWS::ElastiCache::CacheCluster
```
When creating ElastiCache for Redis, you should choose the former when creating two or more nodes including read replica, and the latter when creating only one node.
In this case, we will create one read replica node in addition to the primary node, so we will create a replication group.
```yaml

```
- ElasticCache username and user group
AWS document mentioned that "To add proper access control to a cluster, replace this `default` user with a new one that isn't enabled or uses a strong password.
https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Clusters.RBAC.html#rbac-using
Generate password for user `default` and store it in SecretsManager for latter access from EKS:
```yaml
  CacheUserSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      # Elastic cache authentication
      Name: !Sub ${AWS::StackName}-${usecase}
      Description: "Secret with dynamically generated password."
      GenerateSecretString:
        SecretStringTemplate: !Sub '{"username": "default"}'
        GenerateStringKey: "password"
        PasswordLength: 16
        ExcludeCharacters: '"@/\&'
```
Create ElastiCache `default` with different ID:
```yaml
  ElastiCacheUser:
    Type: AWS::ElastiCache::User
    Properties:
      AccessString: "on ~* +@all" # < access to all available keys and commands.
      Engine: redis
      NoPasswordRequired: false
      Passwords:
        - !Sub "{{resolve:secretsmanager:${CacheUserSecret}::password}}"
      UserId: !Sub ${AWS::StackName}-${usecase}
      UserName: !Sub "{{resolve:secretsmanager:${CacheUserSecret}::username}}"
```
Attach it to the group:
```yaml
  CacheUserGroup:
    DependsOn: ElastiCacheUser
    Type: AWS::ElastiCache::UserGroup
    Properties:
      Engine: redis
      UserGroupId: !Sub ${AWS::StackName}-${usecase}
      UserIds:
        - !GetAtt ElastiCacheUser.UserId
```
## Deploying
```bash
aws --profile dev-sre cloudformation deploy --template-file elasticcache.yaml \
--stack-name elasticcashe-cluster2 \
--parameter-overrides SubnetIda=subnet-0a65a477f719d7ed7 SubnetIdc=subnet-05f5b908cca8e18a7 SubnetIdc=subnet-09eafabb6b042ebeb \
SecurityGroup=sg-09c9b8d8bb4b3be01
```
- Update
```bash
aws --profile dev-sre cloudformation update-stack --template-file elasticcache.yaml \
--stack-name elasticcashe-cluster2 \
--parameter-overrides SubnetIda=subnet-0a65a477f719d7ed7 SubnetIdc=subnet-05f5b908cca8e18a7 SubnetIdc=subnet-09eafabb6b042ebeb \
SecurityGroup=sg-09c9b8d8bb4b3be01
```