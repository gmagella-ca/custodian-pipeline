# Cloud Custodian pipeline
 
This repo will provide you with a cloudformation template to deploy a policy-as-code pipeline to automate your cloud custodian policies deployment. 
This is desirable for many reasons, some of witch is:

* Allowing for code-base practices (peer review, pull request, etc)
* Allowing for consistent approach of deploying (dry-runs, manual approvals, etc)
* Easy audit of policy versioning and etc.

DISCLAIMER:
Use this solution at your own risk. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. 

## TL;DR:
 
1. Deploy Master CF template in an account you'll be calling "CloudCustodianMaster"
2. Deploy Member CF in all accounts you'll call "Member" (will report to master for CloudCustodianPurposes)
3. Create the git repo with the files expected (intructions [here](#FileInstructions))
4. Update mailer settings (get SQS Queue URL and RoleName from CloudFormation)
5. Write your first policy (here are some [examples](https://cloudcustodian.io/docs/aws/examples/index.html) )
6. Push your changes to git and watch the pipeline go. #Success!
7. PLEASE limit your CloudCustodianAdminRole permission, this example repo creates with Admin privileges.

## Setting up the Multi-Account structure for Custodian.   
 
Custodian can be used either in a single account or in a multi-account configuration.  In this set up we are going to have a central **'Master/Security'** account and a number of **'Member'** accounts which will be managed by the Custodian policies.  These will be referred to a Member and Master respectively from this point forward. 
 
The Architecture Diagram is below: 
 
![alt architecture-diagram][architecture] 
 
[architecture]: img/custodian.png 
 
### Note that at this moment, the Guard-duty piece is *not* in place. (Work In-Progress)

The Master account will be used to host: 
- AWS CodeCommit Repository (a git repository) 
- AWS CodeBuild 
- AWS CodePipeline  
- CloudWatch Event Rules built by Custodian 
- Lambda Functions built by Custodian 
- SNS Topic for outputing Code Pipeline events to email or a webhook 
- SQS, SNS and SES services for sending Custodian notfications 
 
Setting up the Master account requires the Cloud Watch Event bus to be confirgured to receive events from other accounts.  This is done via the CloudFormation template [master.yml](cloudformation/master.yml)  (EventBusPolicy resource). 
 
The Member account will forward CloudWatch Events to the Master Account for monitoring.  It will also have a Trust Relationship with the Master Account which will allow the Master Role to assume the role CloudCustodianAdminRole, which has an AWS Managed Administrator Policy attached to it. IMPORTANT: In a production environment, you'd like to limit this role to the functions you've already vetted in your policies.  
 
When an action is performed in the Member account. e.g. a resource is created, this event is forwarded to the Master event bus and matched against CloudWatch Event Rules created by Custodian Policies.  If it is matched the Rule will trigger a Lambda function.  This will STS Assume Role in the Member account using the CloudCustodianAdmin role to make the defined change to the resource.  
 
There are a few things worth noting:

1. This is a single-region deployment that can be extended to work multi-region with easy (instructions below).
2. This automation approach its only suitable for cloudtrail based policies initially (please see considerations below if you would like to use other types of policies, e.g: periodic / config / etc)

### What you'll need before start:
1. Clone this repo into your machine.
2. Identify the Master Account ID. This should be your "Security" account, not your AWS Master Org Account.
3. Identify the Member Accounts you'll want to monitor with Cloudcustodian (Account IDs)
4. Make sure you have admin access to all accounts in question.

# Member Account 
 
This is the account(s) that will be monitored by Custodian. This account needs to send Cloudwatch Events to the Master (Security) Account. We have to also create a Role and policies with the appropriate permissions for the Master/Security Account to STS/AssumeRole and carry the actions out. Here is how you deploy the resources in the Member Account:
 
#### Deploy the CloudFormation Script via the console 
 
1.  Login to the console (or assume a role via STS) of the Member (monitored) account.  Go to CloudFormation 
2.  Create Stack 
3.  Upload a template to Amazon S3 
4.  Choose file [member.yml](cloudformation/member.yml) 
5.  Provide a StackName 
6.  Provide the account number of the Master (Security) account 
7.  Next 
8.  Next 
9.  Tick "I acknowlewdge that AWS CloudFormation might create IAM resources with custom names." 
10. Create 
11. Optional - deploy [rule.yml](cloudformation/rule.yml) in the other regions you want to monitor.
 
NOTE: You can use Cloudformation StackSets to perform this action in all "Member" accounts for you.

#### Deploy the CloudFormation Script via the CLI 
 
Ensure you have the AWS CLI installed and configured.  You will need IAM Access Keys to use the AWS CLI 
 
- [Installing the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) 
- [Configuring the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) 
 
From the directory containing both the CloudFormation Template and Parameters file. Edit the Parameters File and provide the account Id of the Master Account:  
 
```bash 
vi parameters.json 
 
[ 
  { 
    "ParameterKey": "CustodianMasterAccountId", 
    "ParameterValue":"123412341234" 
  }, 
  { 
    "ParameterKey": "CloudWatchEventRuleName", 
    "ParameterValue": "EventForwardingToCentralAccount" 
  } 
] 
``` 
Deploy the stack using the AWS CLI.  Insert the profile name you are using to STS into the relevant account.  Note that this will create a stack with terminiation protection enabled to prevent accidental deletion. 
 
```bash 
aws cloudformation create-stack \ 
--stack-name EventForwardtoMaster \ 
--template-body file://member.yml \ 
--parameters file://member-parameters.json \ 
--capabilities CAPABILITY_NAMED_IAM \ 
--profile custodian \ 
--region us-east-1 \ 
--enable-termination-protection 
``` 
 
# Master Account  
 
## Deploy CloudFormation to build Repository and CodePipeline 
 
In the Master account deploy the CloudFormation Template master.yml either via the console or via the command line.  Use the --profile for your security account. 
 
```bash 
aws cloudformation create-stack \ 
--stack-name CloudCustodian \ 
--template-body file://master.yml \ 
--parameters file://master-parameters.json \ 
--capabilities CAPABILITY_NAMED_IAM \ 
--profile security \ 
--region us-east-1 \ 
--enable-termination-protection 
``` 
 
This will build: 
- A Code Repository called CloudCustodianPolicies 
- SNS Topic for Manual approvals
- Required IAM Roles
- An SQS Queue for the Mailer function
- An S3 Bucket to host the artifacts
 
- A Code Deployment Pipeline called CloudCustodianDeploymentPipeline with the following stages: 
  - Retrive Policies 
  - PolicyValidationStage 
  - PolicyCleanUpStage 
  - PolicyDeploymentStage 
 
# Setting up the Security Engineer workspace:

## Setting-up your local environment with Cloud Custodian 
To prevent packaging conflicts on your local environment it is recommended that you uses a virtual environment.  
 
check you have pip installed  
 
```bash 
pip --version 
``` 
If you don't have pip installed, then you need to install pip first. Assuming you have pip installed you can install virtualenv.  
 
```bash 
pip install virtualenv 
``` 
Change or create a local directory where you want to keep your c7n scripts, this is your local development environment. 
 
```bash 
mkdir custodian_scripts 
cd custodian_scripts 
``` 
Create a virtual environment and activate it in this directory 
```bash 
virtualenv venv 
source venv/bin/activate 
``` 
You can see that the virtual environment has been activated by the (venv)$.  If you ever need to leave the environment type: 
```bash 
deactivate 
``` 
You can now install c7n, confirm the installation and version. 
```bash 
pip install c7n \ 
custodian version 
``` 
 
If it returns a result you have installed custodian on your local environment 
 
``` 
0.8.41.0 
``` 
 
You now have a local development environment for writing your custodian policies.  Writing and testing policies will be covered later. 

## Create Git Credentials to Commit to the Repository 
 
To allow users to commit to the Repository they need ssh keys to be able to commit 
 
The process is outlined in documentation at [Using IAM with CodeCommit: Git Credentials, SSH Keys, and AWS Access Keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_ssh-keys.html?icmpid=docs_iam_console#ssh-keys-code-commit) 
 
- To set up ssh keys on Linux,macOS, Unix [Configure Credentials on Linux, macOS, or Unix](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html) 
- To set up ssh keys on Windows [Setup Steps for SSH Connections to AWS CodeCommit Repositories on Windows](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-windows.html) 
 
## On your Local Machine 
 
In the directory on your local machine where you will store your custodians scripts. Create two directories: 
 
1. policies 
2. c7n_mailer_config 
3. c7-org-policies
 
Theses folders host: 1: The Cloudtrail-based policies you'd like to apply, 2: Configurations for the mailer function and templates and 3: the policies that needs to be deployed as c7n-org, i.e: the policies using mode "periodic".
 
# FileInstructions

Either you can create the expected directory structure like this:
``` 
cd <folder-you-want-to-host-this-repo>
mkdir policies 
mkdir c7n_mailer_config
mkdir old_policies
git init
git add . \ 
git commit -m "commit message" 
``` 
 
Another option (better) is copying the file structure included in this repo : example-policies/
Linux/MAC:
```
cp -r example-policies/ /path/you/want
```
Windows:
```
xcopy c:\custodian-pipeline\example-policies c:\path\you\want
```

# It's recommended the you edit the policies / config files pointing to the correct SQS queue, Slack webBook, emails addresses, etc prior to commit this code into the repo.

Next we connect the local directory to the remote git repository.  For this you need IAM permissions to connect to the remote git repository in AWS CodeCommit.   
 
## CodeCommit 

## INFO: you can use any GIT tool, in this case, you'd remove the created git repo and edit the pipeline and change the "Source" stage.

Details on how to connect to a AWS CodeCommit repository can be found in: 
 
[Connect to an AWS CodeCommit Repository](https://docs.aws.amazon.com/codecommit/latest/userguide/how-to-connect.html) 
 
There are three options: 
 
- https 
- ssh 
- git-remote-codecommit (highly recommended) [instructions](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-git-remote-codecommit.html)
 
You will need the url of the CodeCommit repository which was created by the CloudFormation stack built in the Security Account. 
 
To locate this via the console: 
 
- In CodeCommit 
- Select the CloudCustodianPolicies repository 
- In the top right select Clone URL 
  - select Clone HTTPS 
  - or Clone SSH 
- The url will be copied to your clipboard 
 
Via the AWS CLI: 
 
```bash 
aws codecommit get-repository --repository-name CloudCustodianPolicies  
``` 
 
Now we can add the committed files on our local repository to the remote repository 
 
```bash 
git remote add origin ssh://git-codecommit.{region}.amazonaws.com/v1/repos/CloudCustodianPolicies 
git remote -v 
git push -u origin master 
``` 
 
### Conditional policy for the Master Branch 
 
A policy should be applied to prevent any user with access to the git repository from Merging into the Master Branch, which will initate a build of the pipeline.  Development of the Custodian Policies should be carried out on a branch and then reviewed before being merged into the main branch.  The blog post [Using AWS CodeCommit Pull Requests to request code reviews and discuss code](https://aws.amazon.com/blogs/devops/using-aws-codecommit-pull-requests-to-request-code-reviews-and-discuss-code/) provides a good overview of how to implement this process. 
 
Details on how to restrict users from merging commits into the Master branch can be found in the documentation at: [Limit Pushes and Merges to Branches in AWS CodeCommit](https://docs.aws.amazon.com/codecommit/latest/userguide/how-to-conditional-branch.html) 
 
Example IAM policy to be applied to a user to prevent them merging commits into the master branch without a review.  
 
```json 
{ 
    "Version": "2012-10-17", 
    "Statement": [ 
        { 
            "Effect": "Deny", 
            "Action": [ 
                "codecommit:GitPush", 
                "codecommit:DeleteBranch", 
                "codecommit:PutFile", 
                "codecommit:MergePullRequestByFastForward" 
            ], 
            "Resource": "arn:aws:codecommit:us-east-2:80398EXAMPLE:MyDemoRepo", 
            "Condition": { 
                "StringEqualsIfExists": { 
                    "codecommit:References": [ 
                        "refs/heads/master",  
                     ] 
                }, 
                "Null": { 
                    "codecommit:References": false 
                } 
            } 
        } 
    ] 
} 
``` 
 
# Writing and Deploying CloudCustodian Policies to the Deployment Pipeline 
 
CloudCustodian policies are written in YAML. The [Cloud Custodian Documentation](https://www.capitalone.io/docs/index.html) provides an introduction and example use cases. 
 
This guide supplements this with how to deploy the policies into the AWS CodeCommit repository and how to deploy them into the Master Account. 
 
## Create a new branch on our local development environment and push it to the remote CodeCommit repository 
 
On your local machine in your policies directory create a new branch and commit your new policies into this branch. Branching allow multiple stages of development to co-exist without conflicting with each other. Branch names are customisable, but are often a sprint or project reference. 
 
First, get the latest version of the remote branch in CodeCommit. 
 
```bash 
git pull 
``` 
Create the branch on your local machine and switch into this branch: 
```bash 
git checkout -b [name_of_your_new_branch] 
``` 
Push the branch to CodeCommit: 
```bash 
git push origin [name_of_your_new_branch] 
``` 
When you want to commit something in your branch.  Add -u parameter to set upstream.  If you don't add the -u parameter, you will only commit into your local branch on your local machine. 
 
## Writing our first Custodian Policy and pushing it to the remote repository 
 
Having switched into a new branch, we can start to create a new policy, validate it and add it to the local repository before we push it to the remote repository. 
 
In the policies directory on our local development environment create a file called my-first-policy.yml and add the text below. 
 
```bash 
vi my-first-policy.yml 
 
policies: 
  - name: my-first-policy 
    description: | 
      Stops an ec2 instance when it is created with a tag with a key of Custodian. 
    resource: ec2 
    mode: 
      type: cloudtrail 
      role: arn:aws:iam::{account_id}:role/CloudCustodianAdminRole 
      member-role: arn:aws:iam::{account_id}:role/CloudCustodianAdminRole 
      events:  
        - RunInstances 
    filters: 
      - "tag:Custodian": present 
    actions: 
      - stop 
``` 
 
In the mode section we have specified that this will be event driven from a CloudTrail event, we have also specified the role in the Master Account and also the Role in the Member Account.  The policy uses {account_id} as a variable to run it in the relevant account.   
 
**Note** - For consistency it is recommended that you have a named Role **CloudCustodianAdminRole** in every account to avoid having to customise this across policies. 
 
Now we need to validate if this policy can be used by custodian.  We perform this on our local development environement before we commit it to the remote environment. 
 
```bash 
custodian validate my-first-policy.yml 
 
2019-03-15 18:03:47,133: custodian.commands:INFO Configuration valid: my-first-policy.yml 
``` 
 
Now this needs added or amended file needs to be added to our local repository and commited to the local repository.  This makes a record of the change. We then push this change to our remote branch in CodeCommit 
 
```bash 
git add my-first-policy.yml 
git commit -m "created my first custodian policy" 
git push -u origin my-first-policy 
``` 
If you want to see what branches exist locally an remote use. 
 
```bash 
git branch -a 
``` 
In the CodeCommit Console you can see the branches, both the master branch and our new branch my-first-policy.  If you have followed the steps, the policies directory on master should be empty and the policies directory in my-first-policy should contain our yaml file. 
 
To initiate a build of our pipeline the my-first-policy branch containing the my-first-policy.yml file needs to be merged into the master branch. 
 
In the CodeCommit Console Creat under Code, select 'Create pull request'. Select my-first-policy as the source branch, and make sure that master is selected as the destination. Select Compare 
 
![alt pull_request_screen_shot][pull_request] 
 
[pull_request]: img/pull_request.png 
 
Add a Title and Description(optional) 
 
When comparing changes between the branches, green text shows code which has been added, and red shows code which has been removed.  In this case it is a new file, so there is no comparison to an existing file. 
 
![alt pull_request_changes][pull_request_changes] 
 
[pull_request_changes]: img/pull_request_changes.png 
 
Select Create 
 
This will create the Pull Request.  The next step is to Merge the Pull request into the Master Branch. From the pull requests screen select **Merge**. Confirm that you want to Merge the Pull request.  If you select the option to delete the branch after the Pull request, you should only have one branch, the master branch. 
 
Once the Pull Request has been completed and the remote branch has been merged into the Master Branch we need to tidy up our local development environment. This will contain both branches before the Pull request and the master branch on your local environment will no longer the same as the remote branch, since the pull request has added an extra file.  
 
```bash 
git checkout master 
Switched to branch 'master' 
 
git pull 
Updating ccd102f..d102d0f 
Fast-forward 
 policies/my-first-policy.yml | 13 +++++++++++++ 
 1 file changed, 13 insertions(+) 
 create mode 100644 policies/my-first-policy.yml 
 
 git branch -d my-first-policy 
 Deleted branch my-first-policy (was d102d0f) 
 
git branch -a 
* master 
  remotes/origin/master 
  remotes/origin/my-first-policy 
 
git remote update origin --prune 
Fetching origin 
From ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/CloudCustodianPolicies 
 - [deleted]         (none)     -> origin/my-first-policy 
``` 
 
You do not have to delete the branches on either the remote or your local environment, but it is good practice to create unique branches for each change and merge these into the master branch.  This is called 'Feature Branching', and gives a clear, highly focused purpose to each branch. 
 
We include the last command ```git remote update origin --prune``` so that the reference to the now merged branch on the remote CodeCommit repository is removed also.  
 
## Deploying policies via the CodePipeline 
 
The pipeline has been set up that once our policy has been merged into the policies directory in the CodeCommit repository, it will start the deployment process. This has the following steps. 
 
1. RetrievePoliciesAction - This will pull the new/changed policy from the CodeCommit Repoistory 
2. PolicyValidationStage - This will perform a validation of the policies to ensure they are valid Custodian Policies.  This is the equivalent of running ```custodian validate my-policy.yml``` on your local environment. 
  - Manaual Approval is required to proceed 
3.  PolicyCleanupStage - This will remove policies which have been moved into a directory called old_policies.   
  - Manual Approval is required to proceed 
4.  PolicyDeploymentStage 
  - Manual Approval is required to proceed 
 
### Manual Approvals 
 
Manual Approvals require the approver to provide a comment and either accept or reject an approval request.  The build will not proceed until either has been selected.   
 
![alt approval_request][approval_request] 
 
[approval_request]: img/approval_request.png 
 
 
Step 4 will deploy the necessary infrastructure in AWS required by the Custodian Policy.  This step is the final step from this point on your policy is **LIVE** and could make changes to your infrastructure. 
 
 
## Testing our policy 
 
To test the policy  
 
1.  In a monitored Member account.   
2.  Create an EC2 Instance (any instance size or type) 
3.  Ensure that in the Tags, there is a value 'Custodian' as a Key Value 

## Pro Tips:

1. Implement a tag "contact_tags" on your resources, so c7n-mailer can email the owner of the resources. TODO: Write a how-to configure a policy to do that.

# Multi-Region Set-up

If your AWS Footprint is spanned across multiple regions, these are the changes you'll have to perform to expand this set-up to support it:

## Instructions on how to make this pipeline multi-region:

Change the buildspec for CloudCustodianPolicyDeploymentProject from:
```
custodian run --assume arn:aws:iam::${AWS::AccountId}:role/CloudCustodianAdminRole --output-dir output/logs policies/* 
```
to (add as many regions as you'd like with multiple --region):
```
custodian run --assume arn:aws:iam::${AWS::AccountId}:role/CloudCustodianAdminRole --output-dir output/logs policies/* --region <region1> --region <region2>
```

This will make the deployment phase create the lambda functions in all the regions you need, but we need to forward events to the master account on that region so the cloudwatch event can get triggered. A cloudformation template is provided (rule.yaml).

### create the rule on the child accounts on the regions you want to support:

Use the template rule.yaml to deploy the rules in any aditional region. Use the details exported on the by the MEMBER stack (step 11 on the instructions below)

# Using other kinds of policies like periodic (a.k.a: Using c7n-org):

For increased security, we want all the lambdas be built the Master account and reacting to cloudtrail events flowing through the event bus but this architecture decision can limit our ability to perform certain tasks / policy checks (e.g: periodic checks). In order for this kind of policies to work we need to deploy them on all accounts. Enters [c7n-org](https://www.cloudcustodian.io/docs/tools/c7n-org.html)!

The pipeline have an action called c7n-org_Deploy, this action will run "c7n-org run" command to deploy the policy *c7n-org-policies/organization-policies.yml* using the file *c7n-org-policies/accounts.yml*. 

Note: To create the accounts.yml file, you might want to use the config file generation steps [described in the documentation](https://www.cloudcustodian.io/docs/tools/c7n-org.html#config-file-generation). 

Here is an example of one account on this file.

```
- account_id: '123456789123'
  email: email@example.com
  name: Account-Name
  role: arn:aws:iam::123456789123:role/CloudCustodianAdminRole
  tags:
  - path:/
```

IMPORTANT: In order for the action c7n-org_Deploy to be able to assume the CloudCustodianAdminRole on the member accounts, you'll have to change the policy CloudCustodianPolicyDeploymentProjectRolePolicy and add the permission to assume role on all member accounts, similar to this:

```
{
    "Action": "sts:AssumeRole",
    "Resource": [
        "arn:aws:iam::<master-accound-id>:role/CloudCustodianAdminRole",
        "arn:aws:iam::<member-account1>:role/CloudCustodianAdminRole",
        "arn:aws:iam::<member-account2>:role/CloudCustodianAdminRole"
    ],
    "Effect": "Allow"
},
```

Note: If this file is not present, the dry-run command will skip, but the action "c7n-org" will still provision and run skipping steps, if you do not intend to use this, you might want to remove the action so you won't be paying for resources you're using with no purpose. 

## TO-DO:

1. Improve documentation on c7n-mailer (how to properly set-up)