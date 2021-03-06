---
title: "AWS Parallel Cluster 101"
emoji: "ð»"
type: "tech" # tech: æè¡è¨äº / idea: ã¢ã¤ãã¢
topics: ["HPC", "ParallelCluster"]
published: false
---

## Parallel Cluster ã³ããã¸ã¡

### CLI ãã¤ã³ã¹ãã¼ã«ãã

Parallel Cluster ã® CLI ã¯ Python ã® `virtualenv` ãå©ç¨ãã¦ãããããç¨æããã

```bash
python3 -m pip install --upgrade pip
python3 -m pip install --user --upgrade virtualenv
python3 -m virtualenv ~/hpc-ve
source ~/hpc-ve/bin/activate
```

åã»ã©ä½æããä»®æ³ç°å¢ã«å¥ã£ã¦ãããã¨ãç¢ºèªããã

```bash
which python3
# ~/hpc-ve/bin/python3
# æããå ´åã¯ã`deactivate`
```

ç¶ãã¦ãAWS CLI ãã¤ã³ã¹ãã¼ã«ããã¦ãããã¨ãç¢ºèªãå¥ã£ã¦ããªããã°ãã¤ã³ã¹ãã¼ã«ãã¦ããã

```bash
pip3 install awscli
```

ããã«ãAWS CDK ã«ãä¾å­ãã¦ãããããNode.js ã¨ CDK ã®ã¤ã³ã¹ãã¼ã«ãå¿ããã«ã
[nodenv](https://github.com/nodenv/nodenv#installation) ãªã©ãçµç±ãã¦ã¤ã³ã¹ãã¼ã«ãã¦ããã¨ããã¼ã¸ã§ã³ç®¡çãã§ããã

æå¾ã«ãParallel Cluster ã® CLI ãã¤ã³ã¹ãã¼ã«ããã

```bash
pip3 install aws-parallelcluster
```

AWS ã®è¨­å®ãæ¸ã¾ãã¦ããã¾ãããã

```bash
aws configure
# AWS Access Key ID [None]: YOUR_KEY
# AWS Secret Access Key [None]: YOUR_SECRET
# Default region name [ap-northeast-1]:
# Default output format [JSON]:
```

### è¨­å®ãã¡ã¤ã«ãèªåã§ã¤ãã

```bash
aws ec2 create-key-pair --key-name pcluster-key --query KeyMaterial --output text > ~/.ssh/pcluster-key
```

### Parallel Cluster ã®è¨­å®ãã¡ã¤ã«ãæ¸ã

Slurm ãä½¿ã£ãå¿è¦æä½éã®è¨­å®ãã¡ã¤ã«ã¯ä»¥ä¸ã®éãã«ãªãã¾ãï¼`Region`, `Image`, `HeadNode`, `Scheduling` ã®é ç®ï¼

```yaml
Region: ap-northeast-1
Image:
  Os: alinux2
HeadNode:
  InstanceType: t3.micro
  Networking:
    SubnetId: subnet-02265a025d9462be1
  Ssh:
    KeyName: pcluster-key
Scheduling:
  Scheduler: slurm
  SlurmSettings:
    ScaledownIdletime: 10
  SlurmQueues:
    - Name: queue1
      CapacityType: SPOT
      ComputeResources:
        - Name: t3micro
          InstanceType: t3.micro
          MinCount: 0
          MaxCount: 10
      Networking:
        SubnetIds:
          - subnet-0bcea6a97db79b5ee
      CustomActions:
        OnNodeStart:
          Script: https://test.tgz # s3:// | https://
          Args:
            - arg1
            - arg2
        OnNodeConfigured:
          Script: https://test.tgz # s3:// | https://
          Args:
            - arg1
            - arg2
SharedStorage:
  - MountDir: workspaces
    Name: shared-ngs-resources
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 1200
      DeploymentType: SCRATCH_1
      ExportPath: s3://ngs-data-bucket
      ImportPath: s3://ngs-data-bucket/workspaces
```

#### åºæ¬è¨­å® + Head Node ã®è¨­å®

â» åäººçã«å¿è¦ã ã¨æã£ã¦ããè¨­å®ã®ã¿ãåæãã¦ãã¾ããè©³ç´°ã¯ AWS ã®å¬å¼ãã­ã¥ã¡ã³ããåç§ã®ãã¨ã

- `Region`: AWS ãªã¼ã¸ã§ã³
- `Image`: EC2 ã¤ã³ã¹ã¿ã³ã¹ã® AMI ã®æå ±
  - `Os`: OS ã®ç¨®é¡ï¼`alinux2`, `centos7`, `ubuntu1804`, `ubuntu2004`ï¼
- `HeadNode`: Head Node ã§ä½¿ç¨ãã EC2 ã¤ã³ã¹ã¿ã³ã¹ã®åç¨®è¨­å®
  - `InstanceType`: EC2 ã¤ã³ã¹ã¿ã³ã¹ã¿ã¤ãï¼ä¸åº¦ç«ã¡ä¸ãããæå¾ãæ´æ°ã¯å¹ããªãï¼
  - `Networking`: ãããã¯ã¼ã¯æ§æ
    - `SubnetId`: Head Node ãéç½®ãããµããããã® ID
  - `Ssh`: EC2 ã¤ã³ã¹ã¿ã³ã¹ã«ã¢ã¯ã»ã¹ããããã® SSH æå ±
    - `KeyName`: EC2 ã­ã¼ãã¢å

#### ã¹ã±ã¸ã¥ã¼ã©ï¼Worker Nodeï¼ã®è¨­å®

â» åäººçã«å¿è¦ã ã¨æã£ã¦ããè¨­å®ã®ã¿ãåæãã¦ãã¾ããè©³ç´°ã¯ AWS ã®å¬å¼ãã­ã¥ã¡ã³ããåç§ã®ãã¨ã

- `Scheduling`: ã¹ã±ã¸ã¥ã¼ã©ã®è¨­å®
  - `Scheduler`: ä½¿ç¨ããã¹ã±ã¸ã¥ã¼ã©ï¼`slurm`, `awsbatch`ï¼
  - `SlurmSettings`: Slurm ã®åºæ¬è¨­å®
    `ScaledownIdletime`: ã¸ã§ãããªãå ´åã« node ãçµäºããæéï¼å, ããã©ã«ã: 10 åï¼
  - `SlurmQueues`: slurm ã®ã­ã¥ã¼è¨­å®ï¼slurm ãã¹ã±ã¸ã¥ã¼ã©ã¨ãã¦ä½¿ç¨ãã¦ããå ´åã®ã¿ï¼
    - `Name`: ã­ã¥ã¼ã®ä»»æã®åå
    - `CapacityType`: EC2 ã¤ã³ã¹ã¿ã³ã¹ã®ã­ã£ãã·ãã£ï¼`ONDEMAND`, `SPOT`ï¼
    - `ComputeResources`: ã³ã³ãã¥ã¼ãã£ã³ã°ãªã½ã¼ã¹è¨­å®
      - `name`: ã³ã³ãã¥ã¼ãã£ã³ã°ç°å¢ã®ä»»æã®åå
      - `InstanceType`: EC2 ã¤ã³ã¹ã¿ã³ã¹ã¿ã¤ã
      - `MinCount`: ãªã½ã¼ã¹ã®æå°å¤
      - `MaxCount`: ãªã½ã¼ã¹ã®æå¤§å¤
    - `Networking`: ãããã¯ã¼ã¯è¨­å®
      - `SubnetIds`: ã­ã¥ã¼ãéç½®ãããµããããã® ID
    - `CustomActions`: node ã§å®è¡ããã«ã¹ã¿ã ã¹ã¯ãªãããæå®
      - `OnNodeStart`:

#### å±æã¹ãã¬ã¼ã¸ã®è¨­å® (Amazon FSx for Lustre ã®ã±ã¼ã¹)

- `SharedStorage`: å±æã¹ãã¬ã¼ã¸ã®è¨­å®
  - `MountDir`: å±æã¹ãã¬ã¼ã¸ããã¦ã³ããããã¹
  - `Name`: å±æã¹ãã¬ã¼ã¸ã®åå
  - `StorageType`: å±æã¹ãã¬ã¼ã¸ã®ã¿ã¤ãï¼ãµãã¼ããããå¤ã¯ `Ebs`, `Efs`, `FsxLustre`ï¼
  - `FsxLustreSettings`: FSx for Lustre ã®åç¨®è¨­å®
    - `StorageCapacity`: Lustre ãã¡ã¤ã«ã·ã¹ãã ã® FSx ã®ã¹ãã¬ã¼ã¸å®¹é (`GiB` åä½) ãè¨­å® (1,200 GiB ~)
    - `DeploymentType`: ããã­ã¤ã¿ã¤ãï¼`SCRATCH_1`ã`SCRATCH_2`ã`PERSISTENT_1`ã®ãããããè©³ç´°ã¯å¾è¿°ï¼
    - `ExportPath`: Amazon FSx for Lustre ãã¡ã¤ã«ã·ã¹ãã ã®ã«ã¼ãï¼ã¨ã¯ã¹ãã¼ãåï¼ã¨ãªã S3 ã®ãã¹
    - `ImportPath`: FSx for Lustre ãã¡ã¤ã«ã·ã¹ãã ã®ãã¼ã¿ãªãã¸ããªã¨ãã¦ä½¿ç¨ãã¦ãã S3 ãã±ããã¸ã®ãã¹(`ExportPath` ã¨åä¸ã® S3 ãã±ããã§ããå¿è¦ããã)

### åèæç®

- [Cluster configuration file - AWS Parallel Cluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/cluster-configuration-file-v3.html)
- [example_configs - aws-parallelcluster](https://github.com/aws/aws-parallelcluster/tree/release-3.0/cli/tests/pcluster/example_configs)

## ã«ã¹ã¿ã ã®è¨­å®ãã¤ã³ã¹ã¿ã³ã¹ã«è¿½å ããã

### ã«ã¹ã¿ã  AWS ParallelCluster AMI ã®æ§ç¯

[AWS ã®ãã­ã¥ã¡ã³ã](https://docs.aws.amazon.com/ja_jp/parallelcluster/latest/ug/tutorials_02_ami_customization.html)ã«ãæ¸ãã¦ããã¨ãããã«ã¹ã¿ãã¤ãºã®ããã®ã¢ãã­ã¼ãã¨ãã¦ã«ã¹ã¿ã  AMI ãæ§ç¯ãããã¨ã¯æ¨å¥¨ãã¦ããªãã

çç±ã¨ãã¦ã¯ãä»å¾ã®ãªãªã¼ã¹ã§ã¢ãããã¼ãããã°ä¿®æ­£ãã«ã¹ã¿ã  AMI ã«é©å¿ã§ããªããªãããã¨ããã¦ããï¼ãããããã¢ãããã¼ãçã® AMI ããã¼ã¹ã¨ãã¦ã«ã¹ã¿ã  AMI ãä½ãç´ããªããã°ãªããªãã¨æãããï¼

### ã«ã¹ã¿ã ãã¼ãã¹ãã©ããã¢ã¯ã·ã§ã³ (Custom Bootstrap Actions)

[AWS ã®ãã­ã¥ã¡ã³ã](https://docs.aws.amazon.com/parallelcluster/latest/ug/pre_post_install.html)ã«ããã¨ãã«ã¹ã¿ã ãã¼ãã¹ãã©ããã¢ã¯ã·ã§ã³ãä½¿ããã¨ãæ¨å¥¨ãããæ¹æ³ã®ããã ãã¤ã³ã¹ã¿ã³ã¹ï¼ã¯ã©ã¹ã¿ï¼ããã¼ããããåã¨å¾ã«å¦çãæã¿è¾¼ããï¼pre-install ã¨ post-install ã® 2 ç¨®é¡ï¼

#### pre-install ã¨ post-install ã®å·ä½ä¾

pre-install ã¨ã¯ãNATãAmazon Elastic Block Store (Amazon EBS)ãã¹ã±ã¸ã¥ã¼ã©ã¼ã®è¨­å®ãªã©ããªãããåã®ã¿ã¤ãã³ã°ãæãã

- pre-install ã¢ã¯ã·ã§ã³ä¾
  - ã¹ãã¬ã¼ã¸ã®å¤æ´
  - ã¦ã¼ã¶ã®è¿½å 
  - åç¨®ããã±ã¼ã¸ã»ã©ã¤ãã©ãªã®è¿½å 

post-install ã¨ã¯ãã¤ã³ã¹ã¿ã³ã¹ã®è¨­å®ãå®äºããå¾ã®ã¿ã¤ãã³ã°ãæãã

- post-install ã¢ã¯ã·ã§ã³ä¾
  - ã¹ãã¬ã¼ã¸ã®å¤æ´
  - ã¹ã±ã¸ã¥ã¼ã©ã¼ã®è¨­å®
  - åç¨®ããã±ã¼ã¸ã»ã©ã¤ãã©ãªã®æ´æ°

#### å¯¾å¿ããã¹ã¯ãªãã

`Bash` ã¨ `Python` ã«å¯¾å¿ãã¦ããã

## ãã®ä»

### è¨­å®ãã¡ã¤ã«ã¯ `TOML` ãã `YAML` å½¢å¼ã®ãã¡ã¤ã«ã«ç§»è¡

AWS Parallel Cluster v2.x ç³»ã§ã¯ã`TOML` ã®è¨­å®ãã¡ã¤ã«ãä½¿ã£ã¦ãã¾ããããv3.x ç³»ã§ã¯ `YAML` ãã¡ã¤ã«ã«å¤æ´ããã¦ãã¾ããã¾ã ãå¤ãæå ±ããããä¸ã«ã¯æ®ã£ã¦ããããã`TOML` ã§æ¸ããããµã³ãã«ã®è¨­å®ãã¡ã¤ã«ãå­å¨ãã¾ããããããã¯ v3.x ç³»ã§ã¯å®è¡ã§ããªãã¯ãã§ãã

### ã¹ã±ã¸ã¥ã¼ã©ã¼ã¨ãã¦ `SGE` ã¨ `Torque` ããµãã¼ããããªããªãï¼2021/12/31 ã¾ã§ãµãã¼ãï¼

ç¾è¡ã® AWS Parallel Cluster v3.x ç³»ã§ã¯ã`SGE` ã¨ `Torque` ã®ãµãã¼ãããªããªãã¾ãããçµæã¨ãã¦ãã¹ã±ã¸ã¥ã¼ã©ã¼ã¨ãã¦å©ç¨ã§ããã®ã¯ã`Slurm` ã¨ `AWS Batch` ã® 2 ç¨®é¡ã¨ãªã£ã¦ãã¾ãã

[Configuring AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/getting-started-configuring-parallelcluster.html)

### Parallel Cluster ã®å®è¡ã«å¿è¦ãª IAM ã­ã¼ã«

å¼·åãªæ¨©éãä»ä¸ãã¦å®è¡ãã¦ãè¯ãã§ãããã§ããã ã IAM ã­ã¼ã«ã«å²ãæ¯ãããªã·ã¼ã¯çµãè¾¼ã¿ããã§ããAWS ããããã ãªæ¨©éãç¤ºãã¦ããã¦ããã®ã§ãããããåèã«ã§ãã¾ãã

[AWS Identity and Access Management roles in AWS ParallelCluster 3.x](https://docs.aws.amazon.com/parallelcluster/latest/ug/iam-roles-in-parallelcluster-v3.html)

### Parallel Cluster ã®ãããã¯ã¼ã¯æ§æ

ãã¹ã¦ãããªãã¯ãµããããã§æ§ç¯ããã±ã¼ã¹ã¨ãHead Node ããããªãã¯ãµãããããCompute Node ããã©ã¤ãã¼ããµããããã«åããã±ã¼ã¹ã«å¤§å¥ãããã

ãã®ãããã¯ãè¦ä»¶ãã»ã­ã¥ãªãã£ã¬ãã«ã«å¿ãã¦è¨­å®ãèããå¿è¦ãããã

[Network configurations](https://docs.aws.amazon.com/parallelcluster/latest/ug/network-configuration-v3.html)

### Elastic Fabric Adapter

åããµããããä¸ã®ä»ã®ã¤ã³ã¹ã¿ã³ã¹ã¨ã®**ä½ã¬ã¤ãã³ã·ã¼**ã®ãããã¯ã¼ã¯éä¿¡ãå®ç¾ããããã®ãããã¯ã¼ã¯ããã¤ã¹ã§ãã

Compute Node ã¨ã¹ãã¬ã¼ã¸ããããã¯ã¼ã¯éä¿¡ãã¦ãã¼ã¿ã®ããåããè¡ã£ã¦ããä»¥ä¸ããããã¯ã¼ã¯ã®ã¬ã¤ãã³ã·ã¼ãå¦çä¸ã®ããã«ããã¯ã«ãªãå¯è½æ§ãããã¾ãããã®åé¡ãè§£æ¶ããããã«ãAWS ãç¨æãã¦ãããããã¯ã¼ã¯ããã¤ã¹ã Elastic Fabric Adapter ã§ãã

ãã¹ã¦ã®ã¤ã³ã¹ã¿ã³ã¹ã§å©ç¨å¯è½ãªããã§ã¯ãªããä»¥ä¸ã®ã¤ã³ã¹ã¿ã³ã¹ã§ã®ã¿ä½¿ç¨å¯è½ã§ãã

- c5n.18xlarge
- c5n.metal
- g4dn.metal
- i3en.24xlarge
- i3en.metal
- m5dn.24xlarge
- m5n.24xlarge
- m5zn.12xlarge
- m5zn.metal
- r5dn.24xlarge
- r5n.24xlarge
- p3dn.24xlarge
- c6gn.16xlarge

#### åèæç®

- [Elastic Fabric Adapter - AWS ParallelCluster](https://docs.aws.amazon.com/ja_jp/parallelcluster/latest/ug/efa.html)
- [AWS Elastic Fabric Adapter ã®éä¿¡éåº¦è©ä¾¡](https://tech.preferred.jp/ja/blog/aws-elastic-fabric-adapter-evaluation/)

## Amazon FSx for Lustre ã«ã¤ãã¦

### ãã¡ã¤ã«ã·ã¹ãã 

#### ã¹ã¯ã©ãã (Scratch) ãã¡ã¤ã«ã·ã¹ãã 

ã¹ã¯ã©ãããã¡ã¤ã«ã·ã¹ãã ã§ã¯ãä»¥ä¸ã®ãããªç®çã«ä½¿ããã¾ãã

- ä¸æçãªã¹ãã¬ã¼ã¸ã¨ãã¦ä½¿ããã
- ç­æéã®ãã¼ã¿å¦çã«ä½¿ããã

ãã¡ã¤ã«ã·ã¹ãã ã«éå®³ãçºçãã¦ãããã¼ã¿ã¯è¤è£½ãããä¿æããã¾ããï¼ã¤ã¾ããå¯ç¨æ§ã¯ä½ãã¨ãããã¨ï¼ç­æéã®ã¯ã¼ã¯ã­ã¼ãåããªã®ã§ãéå®³ãçºçãã¦ãã¼ã¿ãæ¶ãã¦ãã¾ã£ã¦ãããã¼ã¿å¦çãããç´ãã°è¯ãã±ã¼ã¹ã«é©ãã¦ãã¾ãã

#### æ°¸ç¶ (Persistent) ãã¡ã¤ã«ã·ã¹ãã 

æ°¸ç¶ãã¡ã¤ã«ã·ã¹ãã ã§ã¯ãä»¥ä¸ã®ãããªç®çã§ä½¿ããã¾ãã

- é·æçã«ä¿æå¯è½ãªã¹ãã¬ã¼ã¸ã¨ãã¦ä½¿ããã
- é·æéã®ãã¼ã¿å¦çã«ä½¿ããã

ãã¼ã¿ã¯ã¢ãã¤ã©ããªãã£ã¼ã¾ã¼ã³ (AZ) åã§èªåè¤è£½ããã¾ãï¼ã¤ã¾ããé«ãå¯ç¨æ§ãæä¿ããã¦ããï¼é·æçã»ç¡æéã«å®è¡ããããã¼ã¿ãéä¸­ã§æ¶ãã¦ãã¾ãã¨å°ãå¦çã«é©ãã¦ãã¾ãã

#### åèæç®

- [Amazon FSx for Lustre ãã¡ã¤ã«ã·ã¹ãã ã®ããã­ã¤ãªãã·ã§ã³ã®ä½¿ç¨ - FSx for Lustre](https://docs.aws.amazon.com/ja_jp/fsx/latest/LustreGuide/using-fsx-lustre.html)

### Lustre ã«ããããã¼ã¿å§ç¸®

Lustre ã®ãã¼ã¿å§ç¸®æ©è½ãå©ç¨ãããã¨ã§ããã¡ã¤ã«ãµã¼ãã¨ã¹ãã¬ã¼ã¸éãã¡ã¤ã«ã·ã¹ãã ãããã¯ã¢ããã¹ãã¬ã¼ã¸ã§ã³ã¹ããåæ¸ã§ãã¾ããã¾ããLustre ãã¡ã¤ã«ãµã¼ãã¨ã¹ãã¬ã¼ã¸ï¼S3 ãªã©ï¼éã§ã®ãã¼ã¿è»¢ééãå°ãããªããããçµæã¨ãã¦ãããã¯ã¼ã¯ã¹ã«ã¼ãããã®æ¹åã«ãç¹ããã¾ãã

ãã¼ã¿å§ç¸®ãæå¹ã«ããã¨ãAmazon FSX for Lustre ã§ã¯ãæ°ããæ¸ãè¾¼ã¾ãããã¡ã¤ã«ããã£ã¹ã¯ã«æ¸ãè¾¼ã¾ããåã«èªåçã«å§ç¸®ãããèª­ã¿è¾¼ã¾ããã¨ãã«èªåçã«è§£åããã¾ãã

ãã¼ã¿å§ç¸®ã§ã¯ãLZ4 ã¢ã«ã´ãªãºã ãä½¿ç¨ãã¦ããããã¡ã¤ã«ã·ã¹ãã ã®ããã©ã¼ãã³ã¹ã¸ã®æªå½±é¿ããªããé«å§ç¸®ã§ããã¨è¬³ããã¦ããï¼ããã©ã¼ãã³ã¹æåã®ã¢ã«ã´ãªãºã ã§ãå§ç¸®éåº¦ãéããé«å§ç¸®ã§ããâ¦ãããï¼

#### åèæç®

- [Lustre data compression - FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/data-compression.html)

### ImportPath ã¨ ExportPath ã«ã¤ãã¦

ã¾ããAmazon FSx for Lustre ã¯èªåçã« **S3 ãã±ãããä½ã£ã¦ããã¾ãã**ããªã®ã§ããã§ã«ä½ææ¸ã¿ã® S3 ãã±ãããæå®ããå¿è¦ãããã¾ãã

ä»¥ä¸ã«å·ä½ä¾ãç¤ºãã¾ãã

- `ImportPath`: ã¤ã³ãã¼ãåã® S3 ãã±ããã®ãã¹
  - ä¾ 1.) s3://import-bucket
  - ä¾ 2.) s3://import-bucket/input-file-
- `ExportPath`: ã¨ã¯ã¹ãã¼ãåã® S3 ãã±ããã®ãã¹
  - ä¾ 1.) s3://import-bucket
  - ä¾ 2.) s3://import-bucket/output

ä¾ 1 ã®ããã«ã`ImportPath` ã¨ `ExportPath` ã«åããã±ãããæå®ãããã¨ãå¯è½ã§ãããã®å ´åãinput ãã¼ã¿ã output ãã¼ã¿ã«ãã£ã¦ä¸æ¸ãããããªã¹ã¯ãããã¨ããç¹ã«æ³¨æã§ãã

`ImportPath` ã®ä¾ 2 ã®ã±ã¼ã¹ã§ã¯ã`input-file-` ã¨ãããã¬ãã£ãã¯ã¹ãã¤ãããã¡ã¤ã«ï¼ãªãã¸ã§ã¯ãï¼ã ããã¤ã³ãã¼ãããã¾ãã

`ExportPath` ã®ä¾ 2 ã®ã±ã¼ã¹ã§ã¯ã`output` ãã£ã¬ã¯ããªã«ãã¡ã¤ã«ãã¨ã¯ã¹ãã¼ãããã¾ãã

#### åèæç®

- [Lustre data compression - S3 ãã±ããã¸ã®ãã¡ã¤ã«ã·ã¹ãã ã®ãªã³ã¯](https://docs.aws.amazon.com/ja_jp/fsx/latest/LustreGuide/create-fs-linked-data-repo.html)

## åèæç®

- [awsdocs - aws-parallelcluster-user-guide](https://github.com/awsdocs/aws-parallelcluster-user-guide)
- [AWS Black Belt Online Seminar - AWS ParallelCluster ã§ã¯ãããã¯ã©ã¦ã HPC](https://d1.awsstatic.com/webinars/jp/pdf/services/20200408_BlackBelt_ParallelCluster.pdf)
- [AWS Black Belt Online Seminar - HPC on AWS](https://d1.awsstatic.com/webinars/jp/pdf/services/20201209_BlackBelt_HPC_on_AWS.pdf)
- [Using cost allocation tags with AWS ParallelCluster](https://aws.amazon.com/jp/blogs/compute/using-cost-allocation-tags-with-aws-parallelcluster/)
- [Monitoring dashboard for AWS ParallelCluster](https://aws.amazon.com/jp/blogs/compute/monitoring-dashboard-for-aws-parallelcluster/)
