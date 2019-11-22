# AMI Provisioning pipeline

## Requirements

- GitHub account
- AWS account

## Generate Github Personal Access Tokens

`Account Settings` --> `Developer Settings` --> `Personal access tokens`

`Generate a new token` with the following scopes:

- repo
  - repo:status
  - repo_deployment
  - public_repo
  - repo:invite 

- admin:repo_hook
  - write:repo_hook
  - read:repo_hook

take note of the generate token, you will need it later when you create the pipeline in `Code Pipeline`.

## Create a GitHub Repository

### AMI Provisioning

The provisioning of the ami will be done via Ansible.

1. create the `requirements.yml` with this content:
    ```yml
    - src: nginxinc.nginx
    - src: rvm.ruby
    ```

2. create the `playbook.yml` with this content:
   ```yml
   ---

   - hosts: all
     become: true
     roles:
       - nginxinc.nginx
       - rvm.ruby
     vars_files:
       - vars/nginx-conf.yml
     vars:
       - rvm1_user: 'root'
     tasks:
       - name: create backend folder
         file:
           path: /usr/etc/website/html
           state: directory
       - name: download codedeploy
         get_url:
           url: https://aws-codedeploy-eu-west-1.s3.amazonaws.com/latest/install
           dest: /tmp
           mode: '0775'
       - name: install codedeploy
         command: /tmp/install auto
   ```
3. create the folder `vars` containing the file `nginx-conf.yml`:
    ```yml
    ---

    nginx_http_template_enable: true
    nginx_http_template:
    default:
        template_file: http/default.conf.j2
        conf_file_name: default.conf
        conf_file_location: /etc/nginx/conf.d/
        port: 80
        server_name: localhost
        error_page: /usr/share/nginx/html
        autoindex: false
        web_server:
        locations:
            default:
            location: /
            html_file_location: /usr/etc/website/html
            html_file_name: index.html
            autoindex: false
        http_demo_conf: false
    ```

### AMI build steps

The build is performed with Packer and orchestrated by CodeBuild.

1. Create the file `packer_cis.json`:
   ```json
   {
    "variables": {
        "vpc": "{{env `BUILD_VPC_ID`}}",
        "subnet": "{{env `BUILD_SUBNET_ID`}}",
        "aws_region": "{{env `AWS_REGION`}}",
        "ami_name": "Prod-CIS-Latest-AMZN-{{isotime \"02-Jan-06 03_04_05\"}}"
    },
    "builders": [{
        "name": "AWS AMI Builder - CIS",
        "type": "amazon-ebs",
        "region": "{{user `aws_region`}}",
        "source_ami_filter": {
        "filters": {
            "virtualization-type": "hvm",
            "name": "CentOS Linux 7 x86_64 HVM EBS ENA*",
            "root-device-type": "ebs"
        },
        "owners": ["679593333241"],
        "most_recent": true
        },
        "instance_type": "t2.micro",
        "ssh_username": "centos",
        "ami_name": "{{user `ami_name` | clean_ami_name}}",
        "tags": {
        "Name": "{{user `ami_name`}}"
        },
        "run_tags": {
        "Name": "{{user `ami_name`}}"
        },
        "run_volume_tags": {
        "Name": "{{user `ami_name`}}"
        },
        "snapshot_tags": {
        "Name": "{{user `ami_name`}}"
        },
        "ami_description": "Amazon Linux CIS with Cloudwatch Logs agent",
        "associate_public_ip_address": "true",
        "vpc_id": "{{user `vpc`}}",
        "subnet_id": "{{user `subnet`}}"
    }],
    "provisioners": [{
        "type": "shell",
        "inline": [
            "sudo yum install -y ansible"
        ]
        },
        {
        "type": "ansible-local",
        "playbook_file": "playbook.yml",
        "host_vars": "vars",
        "playbook_dir": ".",
        "galaxy_file": "requirements.yml"
        },
        {
        "type": "shell",
        "inline": [
            "rm .ssh/authorized_keys ; sudo rm /root/.ssh/authorized_keys"
        ]
        }
    ]
   }
   ```
2. create the file `buildspec.yml`:
    ```yml
    version: 0.2

    phases:
    pre_build:
        commands:
        - echo "Installing Packer"
        - curl -o packer.zip https://releases.hashicorp.com/packer/1.3.3/packer_1.3.3_linux_amd64.zip && unzip packer.zip
        - echo "Validating Packer template"
        - ./packer validate packer_cis.json
    build:
        commands:
        - ./packer build -color=false packer_cis.json | tee build.log
    post_build:
        commands:
        - egrep "${AWS_REGION}\:\sami\-" build.log | cut -d' ' -f2 > ami_id.txt
        # Packer doesn't return non-zero status; we must do that if Packer build failed
        - test -s ami_id.txt || exit 1
        - sed -i.bak "s/<<AMI-ID>>/$(cat ami_id.txt)/g" ami_builder_event.json
        - aws events put-events --entries file://ami_builder_event.json
        - echo "build completed on `date`"
    artifacts:
    files:
        - ami_builder_event.json
        - build.log
    discard-paths: yes
    ```
3. Create the file `ami_builder_event.json`, we will use to retry the `id` of the AMI:
    ```yml
    [
        {
            "Source": "com.ami.builder",
            "DetailType": "AmiBuilder",
            "Detail": "{ \"AmiStatus\": \"Created\"}",
            "Resources": [ "<<AMI-ID>>" ]
        }
    ]
    ```

### Pipeline

In order to glue everything together we are going to use CloudFormation.
We will provision the following services:
- CodePipeline: it will be triggered for all the changes on the repo and it will start a new build via CodeBuild.
- CodeBuild: il will provision a new AMI.
- S3 bucket: it will store the AMI id.

Create the file `pipeline.yml`:
<details>
  <summary>  pipeline.yml </summary>
  <p>

```yml
AWSTemplateFormatVersion: 2010-09-09
Parameters:
  BranchName:
    Description: GitHub branch name
    Type: String
    Default: master
  RepositoryName:
    Description: GitHub repository name
    Type: String
    Default: test
  GitHubOwner:
    Type: String
  GitHubSecret:
    Description: Secret for the webhook
    Type: String
    NoEcho: true
  GitHubOAuthToken:
    Description: Type the token generate at the first point of this guide
    Type: String
    NoEcho: true
  ServiceName:
    Description: Name for this service; used in pipeline names
    Type: String
    Default: HelloWorld
  CodeBuildEnvironment:
    Type: String
    Default: "eb-python-2.6-amazonlinux-64:2.1.6"
    Description: Docker image to use for CodeBuild container - Use http://amzn.to/2mjCI91 for reference
  BuilderVPC:
    Type: AWS::EC2::VPC::Id
    Description: VPC ID that AMI Builder will use to launch temporary resource
  BuilderPublicSubnet:
    Type: AWS::EC2::Subnet::Id
    Description: Public Subnet ID that AMI Builder will use to launch temporary resource

Resources:

  #############
  # Pipeline  #
  #############

  BuildArtifactsBucket:
    Type: AWS::S3::Bucket

  BuildArtifactsBucketPolicy:
    Type: 'AWS::S3::BucketPolicy'
    Properties:
      Bucket: !Ref BuildArtifactsBucket
      PolicyDocument:
        Version: 2012-10-17
        Statement:
          - Sid: DenyUnEncryptedObjectUploads
            Effect: Deny
            Principal: '*'
            Action: 's3:PutObject'
            Resource: !Join 
              - ''
              - - !GetAtt 
                  - BuildArtifactsBucket
                  - Arn
                - /*
            Condition:
              StringNotEquals:
                's3:x-amz-server-side-encryption': 'aws:kms'
          - Sid: DenyInsecureConnections
            Effect: Deny
            Principal: '*'
            Action: 's3:*'
            Resource: !Join 
              - ''
              - - !GetAtt 
                  - BuildArtifactsBucket
                  - Arn
                - /*
            Condition:
              Bool:
                'aws:SecureTransport': false
  
  CodeBuildServiceRole:
    Type: AWS::IAM::Role
    Properties:
      Path: "/managed/"
      ManagedPolicyArns:
        - "arn:aws:iam::aws:policy/PowerUserAccess"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Action: "sts:AssumeRole"
            Effect: Allow
            Principal:
              Service:
                - codebuild.amazonaws.com
      Policies:
        - PolicyName: CodeBuildAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Sid: "CodeBuildToCWL"
                Effect: Allow
                Action:
                  - "logs:CreateLogGroup"
                  - "logs:CreateLogStream"
                  - "logs:PutLogEvents"
                Resource:
                  - !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/codebuild/${ServiceName}_build"
                  - !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/codebuild/${ServiceName}_build:*"
              - Sid: "CodeBuildToS3ArtifactRepo"
                Effect: Allow
                Action:
                  - "s3:GetObject"
                  - "s3:GetObjectVersion"
                  - "s3:PutObject"
                Resource: !Sub "arn:aws:s3:::${BuildArtifactsBucket}/*"

  CodeBuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Sub "${ServiceName}_AMI_provisioning_build"
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: !Sub "aws/codebuild/${CodeBuildEnvironment}"
        EnvironmentVariables:
          - Name: BUILD_OUTPUT_BUCKET
            Value: !Ref BuildArtifactsBucket
          - Name: BUILD_VPC_ID
            Value: !Ref BuilderVPC
          - Name: BUILD_SUBNET_ID
            Value: !Ref BuilderPublicSubnet
      ServiceRole: !GetAtt CodeBuildServiceRole.Arn
      Source:
        Type: CODEPIPELINE

  PipelineWebhook:
    Type: 'AWS::CodePipeline::Webhook'
    Properties:
      Authentication: GITHUB_HMAC
      AuthenticationConfiguration:
        SecretToken: !Ref GitHubSecret
      Filters:
        - JsonPath: $.ref
          MatchEquals: 'refs/heads/{Branch}'
      TargetPipeline: !Ref Pipeline
      TargetAction: SourceAction
      Name: PipelineWebhook
      TargetPipelineVersion: !GetAtt 
        - Pipeline
        - Version
      RegisterWithThirdParty: true

  PipelineExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      Path: "/managed/"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Action: "sts:AssumeRole"
            Effect: Allow
            Principal:
              Service:
                - codepipeline.amazonaws.com
      Policies:
        - PolicyName: CodePipelineS3ArtifactAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Action:
                  - "s3:GetObject"
                  - "s3:GetObjectVersion"
                  - "s3:GetBucketVersioning"
                  - "s3:PutObject"
                Effect: Allow
                Resource:
                  - !Sub "arn:aws:s3:::${BuildArtifactsBucket}"
                  - !Sub "arn:aws:s3:::${BuildArtifactsBucket}/*"
        - PolicyName: CodePipelineBuildAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Action:
                  - "codebuild:StartBuild"
                  - "codebuild:StopBuild"
                  - "codebuild:BatchGetBuilds"
                Effect: Allow
                Resource: !GetAtt CodeBuildProject.Arn
                
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      ArtifactStore:
        Location: !Ref BuildArtifactsBucket
        Type: S3
      Name: !Sub ${ServiceName}_AMI_provisioning_pipeline
      RoleArn: !GetAtt PipelineExecutionRole.Arn
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: ThirdParty
                Version: 1
                Provider: GitHub
              OutputArtifacts:
                - Name: SourceZip
              Configuration:
                Owner: !Ref GitHubOwner
                Repo: !Ref RepositoryName
                Branch: !Ref BranchName
                OAuthToken: !Ref GitHubOAuthToken
                PollForSourceChanges: false
              RunOrder: 1
        - Name: Build
          Actions:
            - Name: CodeBuild
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: 1
              Configuration:
                ProjectName: !Ref CodeBuildProject
              InputArtifacts:
                - Name: SourceZip
              OutputArtifacts:
                - Name: BuiltZip
              RunOrder: 2
Outputs:
  ArtifactRepository:
    Description: S3 Bucket for Pipeline and Build Artifacts
    Value: !Ref BuildArtifactsBucket

  CodePipelineServiceRole:
    Description: CodePipeline IAM Service Role
    Value: !GetAtt PipelineExecutionRole.Arn
```
</p>
</details>

## Setup the pipeline

### TODO insstruction about how to run the CloudFormation