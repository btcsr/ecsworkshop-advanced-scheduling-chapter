#          

# Advanced Scheduling techniques on ECS

## 1. Introduction

In this walkthrough, we are going to discuss what happens when an Amazon ECS task that uses the EC2 launch type is launched.  Amazon ECS would determine where to place the task based on the **requirements** specified in the  ECS task definition, such as CPU and memory. Additionally, we are going to use two types of ECS optimized AMI (ARM and GPU) and register the container instances to the same ECS Cluster to learn how to place task(s) on the desired container instance based on the AMI architecture.

When Amazon ECS places task(s), it uses the following process to select the desired container instance:

1. Identifies the instance that satisfies the CPU, Memory and port requirements in that task definition
2. Identifies the instances that satisfies the task [placement constraints](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-constraints.html). 
3. Identifies the instances that satisfies the task [placment strategies](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-strategies.html).
4. Selects the instances for task placement. 


With the above knowledge, let’s get started with a walk through of ECS task placement.

## 2. What is a Capacity Provider ?

We are going to use ECS Cluster with ECS Capacity Provider. Amazon ECS Capacity providers are used to manage the infrastructure the tasks in your clusters use. Each cluster can have one or more capacity providers and an optional default capacity provider strategy. The capacity provider strategy determines how the tasks are spread across the cluster's capacity providers. When you run a standalone task or create a service, you may either use the cluster's default capacity provider strategy or specify a capacity provider strategy that overrides the cluster's default strategy.


## Code Review

### 3. Navigate to the platform repo

```
$ cd ~/environment/ecsworkshop/
```

We are defining our deployment configuration via code. Let’s look through the code to better understand how the CloudFormation stack is going to create resources. 

### 3.2 Deploying Networking CFN Stack:

Create standard networking resources (VPC, Public and Private Subnets) using the following AWS CloudFormation (CFN) template, naming the stack as `ecsworkshop-vpc`:

```
$ aws cloudformation create-stack \
    --stack-name=ecsworkshop-vpc \
    file://ecsworkshop-vpc.yaml \
    --capabilities CAPABILITY_NAMED_IAM
```

As a part of creating the networking infrastructure, we will be creating a VPC and a couple of Public and Private Subnets using the below:
[Image: Screen Shot 2021-04-23 at 7.41.22 PM.png]
Once the above CFN stack is ready (reached CREATE_COMPLETE state), the stack will export values namely `VPCId`, `SecurityGroup` , `Public & Private SubnetIds`. We will need these details to create the ECS Container Instances. Following is the CFN `DescribeStack` API’s output, which verifies the creation of networking resources:

```
$ aws cloudformation describe-stacks --stack-name ecsworkshop-vpc --query 'Stacks[*].Outputs' --output table
-------------------------------------------------------------------------------------------------------------------------------------
|                                                          DescribeStacks                                                           |
+--------------------+------------------------------------+-------------------+-----------------------------------------------------+
|     Description    |            ExportName              |     OutputKey     |                     OutputValue                     |
+--------------------+------------------------------------+-------------------+-----------------------------------------------------+
|  Private Subnets   |  ecsworkshop-vpc-PrivateSubnetIds  |  PrivateSubnetIds |  subnet-056b33da478b11f58,subnet-07a7ef29d05b2ada5  |
|  Public Subnets    |  ecsworkshop-vpc-PublicSubnetIds   |  PublicSubnetIds  |  subnet-00f53ccb59fba3013,subnet-075535e0065bee628  |
|  ECS Security Group|  ecsworkshop-vpc-SecurityGroups    |  SecurityGroups   |  sg-06767fd76e0421a4e                               |
|  The VPC Id        |  ecsworkshop-vpc-VpcId             |  VpcId            |  vpc-074d610367de94848                              |
+--------------------+------------------------------------+-------------------+-----------------------------------------------------+
```

### 3.3 Deploying the Cluster Resources:

Now, let’s create the ECS Cluster infrastructure, using the following command. In this stack deployment, we are importing the VPC, Security Group, and Subnets from the VPC base platform stack. 

```
aws cloudformation create-stack \
    --stack-name=ecs-demo \
    --template-body=ecsworkshop-demo.yaml \
    --capabilities CAPABILITY_NAMED_IAM
```

The above command creates the following AWS resources: ECS Cluster, Launch Configuration,  AutoScaling Groups. We are going to create two AutoScaling Groups (ARM64 and GPU based).

To retrieve an Amazon ECS-optimized Amazon Linux 2 (arm64) AMI manually, following is the AWS SSM CLI command:

```
aws ssm get-parameters —names /aws/service/ecs/optimized-ami/amazon-linux-2/arm64/recommended
```

To retrieve an Amazon ECS GPU-optimized AMI manually, following is the AWS CLI command:

```
aws ssm get-parameters —names /aws/service/ecs/optimized-ami/amazon-linux-2/gpu/recommended
```

### **3.4 ARM based task deployment code**

For the ARM based capacity provider in the ECS cluster, there are quite a few resources that have to be created to start Container resources to run ARM based tasks. Those resources are the ECS Cluster, AutoScaling Group, Capacity Provider and the Task Definition with ARM based docker image configuration and PlacementConstraints configuration.  

```
Resources:
# Shared ECS Cluster. 
  ECSCluster:
    Type: AWS::ECS::Cluster

# ARM64 based Launch Configuration. 
  ArmASGLaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: !Ref ArmLatestAmiId
      SecurityGroups: !Split
        - ','
        - Fn::ImportValue: !Sub "${VPCStackParameter}-SecurityGroups"
      InstanceType: !Ref 'ArmInstanceType'
      IamInstanceProfile: !Ref 'EC2InstanceProfile'
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          echo ECS_CLUSTER=${ECSCluster} >> /etc/ecs/ecs.config
          yum install -y aws-cfn-bootstrap
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource ArmECSAutoScalingGroup --region ${AWS::Region}

 # AutoScalingGroup to launch Container Instances using ARM64 Launch Configuration.  
  ArmECSAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      NewInstancesProtectedFromScaleIn: true
      VPCZoneIdentifier: !Split
        - ','
        - Fn::ImportValue: !Sub "${VPCStackParameter}-PrivateSubnetIds"
      LaunchConfigurationName: !Ref 'ArmASGLaunchConfiguration'
      MinSize: '0'
      MaxSize: !Ref 'MaxSize'
      DesiredCapacity: !Ref 'DesiredCapacity'
      Tags:
        - Key: Name
          Value: !Sub 'ARM64-${ECSCluster}'
          PropagateAtLaunch: true
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
    UpdatePolicy:
      AutoScalingReplacingUpdate:
        WillReplace: true

#Capacity Provider configuration to create CapacityProvider for ARM64 ASG. Capacity Provider needs ARM ASG Arn, 
# so CloudFormation customer resource ARMClusterResource will make describe API call to ARM ASG to get the desired value. 
  ArmECSCapacityProvider:
    Type: AWS::ECS::CapacityProvider
    Properties:
        AutoScalingGroupProvider:
            AutoScalingGroupArn: !Select [0, !GetAtt ARMCustomResource.Data ]
            ManagedScaling:
                MaximumScalingStepSize: 10
                MinimumScalingStepSize: 1
                Status: ENABLED
                TargetCapacity: 100
            ManagedTerminationProtection: ENABLED

# ECS Task Definition for ARM64 Instance type. PlacementConstraints properties are setting the desired cpu-architecture to arm64.
  Arm64taskdefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Join ['', [!Ref 'AWS::StackName', -arm64]]
      PlacementConstraints: 
        - Type: memberOf
          Expression: 'attribute:ecs.cpu-architecture == arm64'
      ContainerDefinitions:
      - Name: simple-arm64-app
        Cpu: 10
        Command:
          - sh
          - '-c'
          - 'uname -a'
        Essential: true
        Image: public.ecr.aws/amazonlinux/amazonlinux:latest
        Memory: 200
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group: !Ref 'CloudwatchLogsGroup'
            awslogs-region: !Ref 'AWS::Region'
            awslogs-stream-prefix: ecs-arm64-demo-app
```

### **3.5 GPU based task deployment code**

Likewise, for GPU base capacity provider in ECS cluster, there are quite a few resources that have to be created to start the Container resources to run GPU base task. Those resources are AutoScaling Group, Capacity Provider and Task Definition with GPU based docker image configuration and PlacementConstraints configuration.  

```
# GPU based Launch Configuration. 
  GpuASGLaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: !Ref GPULatestAmiId
      # SecurityGroups: [!Ref 'EcsSecurityGroup']
      SecurityGroups: !Split
        - ','
        - Fn::ImportValue: !Sub "${VPCStackParameter}-SecurityGroups"
      InstanceType: !Ref 'GpuInstanceType'
      IamInstanceProfile: !Ref 'EC2InstanceProfile'
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          echo ECS_CLUSTER=${ECSCluster} >> /etc/ecs/ecs.config
          yum install -y aws-cfn-bootstrap
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource GpuECSAutoScalingGroup --region ${AWS::Region}

# AutoScalingGroup to launch Container Instances using GPU Launch Configuration.  
  GpuECSAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      NewInstancesProtectedFromScaleIn: true
      VPCZoneIdentifier: !Split
        - ','
        - Fn::ImportValue: !Sub "${VPCStackParameter}-PrivateSubnetIds"
      LaunchConfigurationName: !Ref 'GpuASGLaunchConfiguration'
      MinSize: '0'
      MaxSize: !Ref 'MaxSize'
      DesiredCapacity: !Ref 'DesiredCapacity'
      Tags:
        - Key: Name
          Value: !Sub 'GPU-${ECSCluster}'
          PropagateAtLaunch: true
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
    UpdatePolicy:
      AutoScalingReplacingUpdate:
        WillReplace: true

#Capacity Provider configuration to create CapacityProvider for GPU ASG. Capacity Provider needs GPU ASG Arn, 
# so CloudFormation customer resource ARMClusterResource will make describe API call to GPU ASG to get the desired value. 
  GpuECSCapacityProvider:
    Type: AWS::ECS::CapacityProvider
    Properties:
        AutoScalingGroupProvider:
            AutoScalingGroupArn: !Select [1, !GetAtt ARMCustomResource.Data ]
            ManagedScaling:
                MaximumScalingStepSize: 10
                MinimumScalingStepSize: 1
                Status: ENABLED
                TargetCapacity: 100
            ManagedTerminationProtection: ENABLED
  
# ECS Task Definition for GPU Instance type. PlacementConstraints properties are setting the desired cpu-architecture to gpu.
  Gputaskdefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Join ['', [!Ref 'AWS::StackName', -gpu]]
      PlacementConstraints: 
        - Type: memberOf
          Expression: !Sub 'attribute:ecs.instance-type == ${GpuInstanceType}'
      ContainerDefinitions:
      - Name: simple-gpu-app
        Cpu: 100
        Essential: true
        Image: nvidia/cuda:11.0-base
        Memory: 80
        ResourceRequirements:
          - Type: GPU
            Value: '1'
        Command:
          - sh
          - '-c'
          - nvidia-smi
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group: !Ref 'CloudwatchLogsGroup'
            awslogs-region: !Ref 'AWS::Region'
            awslogs-stream-prefix: ecs-gpu-demo-app
```

### 3.6 Capacity Provider Association with ASG

A capacity provider must be associated with a cluster prior to being specified in a capacity provider strategy. So we have to associate the ARM and GPU based ECS Capacity providers with the ECS cluster. 

When multiple capacity providers are specified within a capacity provider strategy, at least one of the capacity providers must have a weight value greater than zero. Any capacity providers with a weight of `0` will not be used to place tasks. If you specify multiple capacity providers in a strategy that all have a weight of `0`, any `RunTask` or `CreateService` actions using the capacity provider strategy will fail.

By configuring Weight:1 for GPU Capacity provider, it will be default capacity provider for this ECS cluster. 

```
#Associate ECS Cluster Capacity Provider with both the ARM and CPU capacity provider. 
  ClusterCPAssociation:
    Type: "AWS::ECS::ClusterCapacityProviderAssociations"
    Properties:
      Cluster: !Ref ECSCluster
      CapacityProviders:
        - !Ref ArmECSCapacityProvider
        - !Ref GpuECSCapacityProvider
      DefaultCapacityProviderStrategy:
        - Base: 0
          Weight: 0
          CapacityProvider: !Ref ArmECSCapacityProvider
        - Base: 0
          Weight: 1
          CapacityProvider: !Ref GpuECSCapacityProvider
```

As a part of the above CFN Stack creation, we are creating AWS Custom Resources to obtain both (ARM64 & GPU) the AutoScalingGroup ARNs. We also create the IAM resources: ECS Service Role, AutoScaling Group Service Role, Lambda Execution Role for custom resources and Instance Role for ECS container instances. 


The following is the output which shows successful creation of Cluster Resources in your account:

```
$ aws cloudformation describe-stacks --stack-name ecs-demo --query 'Stacks[*].Outputs' --output table
------------------------------------------------------------------------------------------------------------------------------
|                                                       DescribeStacks                                                       |
+------------------------+-------------------------+-------------------------------------------------------------------------+
|       Description      |        OutputKey        |                               OutputValue                               |
+------------------------+-------------------------+-------------------------------------------------------------------------+
|  ECS Cluster name      |  ecscluster             |  ecs-demo-ECSCluster-NPsCvf3k6aWv                                      |
|  Arm Capacity Provider |  ArmECSCapacityProvider |  ecs-demo-ArmECSCapacityProvider-FXpQH4EJ6DSz                          |
|  GPU Capacity Provicder|  GpuECSCapacityProvider |  ecs-demo-GpuECSCapacityProvider-KYleqfF16Iqy                          |
|  Arm64 task definition |  Armtaskdef             |  arn:aws:ecs:us-west-2:012345678912:task-definition/ecs-demo-arm64:3   |
|  GPU task definition   |  Gputaskdef             |  arn:aws:ecs:us-west-2:012345678912:task-definition/ecs-demo-gpu:2     |
+------------------------+-------------------------+-------------------------------------------------------------------------+
```

That’s it. We have deployed the base platform. Now, let’s move on to deploying ECS Tasks using desired task definitions in the shared ECS cluster. 


## 4. Verify the created resources:

To run a ECS task on an ARM based EC2 instance, we need to provide three input parameters to the RunTask cli option. The RunTask command need ECS cluster, Task definition and CapacityProvider strategy. We generate the ARM_TASKDEF shell environment using the CloudFormation Output value. 

When we create the ARM based task definition using CloudFormation, we configured the task placement constraints. According to our placement constraint configuration, ECS scheduler will take the container instance CPU architecture and task will be placed only if CPU architecture is arm64 or else task will not be placed on Container instances with the ECS cluster. 

We can confirm the placement constraint configuration by describing the ARM based task definition and query the output for taskDefinition.placementConstraints value. This command also confirms that our ARM_TASKDEF value is set correctly.

```
$ ARM_TASKDEF=$(aws cloudformation describe-stacks --stack-name ecs-demo \
    --query 'Stacks[*].Outputs[?OutputKey==`Armtaskdef`]' --output text | awk '{print $NF}')
    
$ aws ecs describe-task-definition --task-definition $ARM_TASKDEF \
    --query 'taskDefinition.placementConstraints' 
[
    {
        "type": "memberOf",
        "expression": "attribute:ecs.cpu-architecture == arm64"
    }
]
```

Likewise, configured GPU_TASKDEF environment variable by querying the CloudFormation stack output for GPU task definition resource name.  We also created GPU base task definition and while creating this task definition, we configured the placement configuration to look for an instance-type equals to p2.xlarge. We confirm this by executing the describe task definition command against GPU_TASKDEF. 

```
$ GPU_TASKDEF=$(aws cloudformation describe-stacks --stack-name ecs-demo \
 --query 'Stacks[*].Outputs[?OutputKey==`Gputaskdef`]' --output text | awk '{print $NF}')
 
$ aws ecs describe-task-definition --task-definition $GPU_TASKDEF \
     --query 'taskDefinition.placementConstraints'
[
    {
        "type": "memberOf",
        "expression": "attribute:ecs.instance-type == p2.xlarge"
    }
]
```

Based on these constraints, when a user tries to launch an ECS Task (CreateService/RunTask), the ECS scheduler will look for the `constraints` field in the task definition. Based on the constraints, the task will be placed on the Container Instance within the ECS Cluster or whichever fulfills the requirement.

If none of the available Container Instance(s) fulfill the requirement, the ECS scheduler will not be able to place the task.

You can configure ECS Constraints in ECS Task Definition using both [built-in](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-constraints.html#attributes)and [custom attributes](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-constraints.html#add-attribute).

## 5. Default Capacity Provider for ECS Cluster

Based on the Capacity Provider Weight value, ECS Cluster will take it as the default Capacity Provider for the ECS cluster. Use the following command to verify the weight:

```
$ ECS_CLUSTER=$(aws cloudformation describe-stacks --stack-name ecs-demo \
    --query 'Stacks[*].Outputs[?OutputKey==`ecscluster`]' --output text | awk '{print $NF}')

$ aws ecs describe-clusters --cluster $ECS_CLUSTER --query 'clusters[].defaultCapacityProviderStrategy' --output table----------------------------------------------
|              DescribeClusters              |
+------+--------------------------+----------+
| base |    capacityProvider      | weight   |
+------+--------------------------+----------+
|  0   |  ArmECSCapacityProvider  |  0       |
|  0   |  GpuECSCapacityProvider  |  1       |
+------+--------------------------+----------+
```

The *base* value designates how many tasks, at a minimum, to run on the specified capacity provider. Only one capacity provider in a capacity provider strategy can have a base defined.

The *weight* value designates the relative percentage of the total number of launched tasks that should use the specified capacity provider. For example, if you have a strategy that contains two capacity providers, and both have a weight of `1`, then when the base is satisfied, the tasks will be split evenly across the two capacity providers. Using that same logic, if you specify a weight of `1` for *capacityProviderA* and a weight of `4` for *capacityProviderB*, then for every one task that is run using *capacityProviderA*, four tasks would use *capacityProviderB*.

## 6. Task Placement Validation

### 6.1 List of instances.

List container instances before you submit tasks to make sure you don’t have any container instances. And after you submit tasks, the capacity provider should launch an EC2 instance and register it to an ECS cluster to run the task


```
$ aws ecs list-container-instances --cluster $ECS_CLUSTER --output table
+----------------------+
|ListContainerInstances|
+----------------------+
```

### 6.2 Validation under success scenario

By default, when you place task using RunTask , the ECS service scheduler will select the default ECS Capacity Provider with weight 1. If you want to submit a job to a desired Capacity Provider, you need to pass it as an input parameter with the RunTask cli option. We have also configured a constraint, so the ECS cluster will check for constraints before it places the task on desired instance. 


```
$ GPU_ECSCP=$(aws cloudformation describe-stacks --stack-name ecs-demo \
    --query 'Stacks[*].Outputs[?OutputKey==`GpuECSCapacityProvider`]' --output text | awk '{print $NF}')
$ aws ecs run-task --cluster $ECS_CLUSTER --task-definition $GPU_TASKDEF --capacity-provider-strategy "capacityProvider=${GPU_ECSCP}"
```



```
$ ARM_ECSCP=$(aws cloudformation describe-stacks --stack-name ecs-demo \
    --query 'Stacks[*].Outputs[?OutputKey==`ArmECSCapacityProvider`]' --output text | awk '{print $NF}')
$ aws ecs run-task --cluster $ECS_CLUSTER --task-definition $ARM_TASKDEF --capacity-provider-strategy "capacityProvider=${ARM_ECSCP}"
```

You will get a JSON response indicating that a Task was submitted. Check for failure. If it is empty it means the task was submitted successfully.  Also look for task lastStatus filed in JSON output and is in PROVISIONING stage. 

Wait for couple of minutes until the cluster autoscaling utilization metric of both the Capacity provider is triggered and,  ARM and GPU based AutoScaling is scaled out.  

If you go back to the ECS console and select the ecs-demo-ECSCluster-RANDOM, you will see that the Capacity provider ArmECSCapacityProvider and GpuECSCapacityProvider Current size increased to 1 for each. 

[Image: Screen Shot 2021-05-14 at 10.07.56 PM.png]If you go to the ECS Instance tab, you will see that now you have two EC2 instances 1 for each task. On ECS Instance tab, click on settings icon and select ecs.cpu-arhitecture and ecs-instance-type. Based on this value, you will see that the Capacity provider launched both GPU (instance-type=p2.xlarge) and ARM (cpu-architecture=arm64) instance types as required for the respective task definition constraints. 

[Image: Screen Shot 2021-05-14 at 10.17.21 PM.png]
### 6.3 Capacity provider will terminate the instances after 15 minutes after tasks are stopped successfully. 

```
$ aws ecs list-container-instances --cluster $ECS_CLUSTER --output table
+----------------------+
|ListContainerInstances|
+----------------------+
```

### 6.4 Validation under failure scenario 

For this ECS cluster, the default Capacity Provider is a GPU based capacity provider. If you execute run-task cli, with $GPU_TASKDEF without ‘—capacity-provider-strategy’ option, the ECS scheduler will take the default capacity provider and will run the job. As all of the constraints are fulfilled, the task will be placed on a container instance and it will complete successfully. 


```
$ aws ecs run-task --cluster $ECS_CLUSTER --task-definition $GPU_TASKDEF 
```

But if you try to run $ARM_TASKDEF without ‘—capacity-provider-strategy ‘ flag, it will get stuck in the PROVISIONING stage. The ECS scheduler will select GPU capacity provider and as $ARM_TASKDEF constraints are not fulfilled, the task will be stuck in the pending state and it will FAIL after 30 minutes. 


```
$ aws ecs run-task --cluster $ECS_CLUSTER --task-definition $ARM_TASKDEF 
```

Wait for a couple of minutes until the cluster autoscaling utilization metric is triggered. Only one instance is launched for GPU based capacity provider as it is the default capacity provider. 

[Image: Screen Shot 2021-05-14 at 10.33.41 PM.png]If you go to the task tab, you will see that, we have only one task in the RUNNING state and another in PROVISIONING. Looking at the task definition, you will see that only the gpu based task is RUNNING and the ARM64 based task is stuck in the PROVISIONING state. 

GPU based task → constraints (instance-type=p2.xlarge) → default capacity provider (GPU) → As both conditions match, the task will be placed successfully. 

ARM base task → constaints(cpu-architecture=arm64) → default capacity provide(GPU) → as architecture of GPU is x86_64 and does not match to arm64, the ECS scheduler will trigger ECS capacity provider to launch the GPU instance type. 

### 6.5 Capacity provider will terminate the instances after 15 minutes after tasks are stopped successfully. 

```
$ aws ecs list-container-instances --cluster $ECS_CLUSTER --output table
+----------------------+
|ListContainerInstances|
+----------------------+
```

## 7. Clean Up


Delete the CloudFormation stack created for this workshop.

```
$ aws cloudformation delete-stack --stack-name ecs-demo
$ aws cloudformation delete-stack --stack-name ecsworkshop-vpc

```


# ecsworkshop-advanced-scheduling-chapter
