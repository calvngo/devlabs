# Automate Setting AWS Account Password Policy for IAM Users #

## Lab Introduction ##
This lab walks you through the steps you need to take to automatically set the IAM Password Policy for all AWS IAM users according to [AWS Foundational Security Best Practices](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-standards-fsbp-controls.html#fsbp-iam-7) standard, which recommends that password policies for IAM users should have strong configurations.<br/>

To access the AWS Management Console, IAM users need passwords. As a best practice, AWS highly recommends that instead of creating IAM users, you use federated access. Federation allows users to use their existing corporate credentials to log into the AWS Management Console. Use [AWS Single Sign-On (AWS SSO)](https://aws.amazon.com/single-sign-on/) to create or federate the user, and then assume an IAM role into an account. After federation activation, you may need to enforce expiration of old user passwords. In this case, this lab will show you how you can effectively update the password policy for all IAM users across all AWS accounts.<br/>

By default, the IAM Password Policy enforces a limited number of conditions:<br/>
 - Minimum password length of 8 characters and a maximum length of 128 characters.
 - Minimum of three of the following mix of character types: uppercase, lowercase, numbers, and ! @ # $ % ^ & * ( ) _ + - = [ ] { } | ' symbols.
 - Not be identical to your AWS account name or email address.

This lab will show you how to update the IAM Password Policy by setting a strong password configuration with the following conditions:<br/>
 - At least one uppercase letter is required.
 - At least one lowercase letter is required.
 - At least one non-alphanumeric character is required.
 - At least one number is required.
 - Minimum password length of 8 characters.

## Lab Design ##
![Screenshot](https://github.com/aws-samples/devlab-iam-password-policy/blob/main/images/Architecture.jpg)<br/>
In this lab, you will deploy an [AWS CloudFormation](https://aws.amazon.com/cloudformation/resources/templates/) template to automatically set the account password policy. The CloudFormation template defines a [custom resource](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html) which invokes an [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) function to perform the required action. The Lambda function  uses the [AWS SDK for Python (Boto3)](https://aws.amazon.com/sdk-for-python/) to call the [AWS Identity and Access Management API](https://docs.aws.amazon.com/IAM/latest/APIReference/API_UpdateAccountPasswordPolicy.html) to update the password policy settings for the AWS account.<br/>

For your reference, the Lambda function runtime is python 3.9 and it requires permission to call the IAM action, UpdateAccountPasswordPolicy, to execute the piece of code below.<br/>
```
import boto3

iam = boto3.client('iam')

response = iam.update_account_password_policy(
   # set parameters as required by AWS FSBP standard control (IAM.7)
   # parameters that are not specified revert to their default values
   RequireNumbers = True,              # must contain at least one numeric character (0 to 9)
   RequireSymbols = True,              # at least one of the characters: ! @ # $ % ^ & * ( ) _ + - = [ ] { } | '
   MinimumPasswordLength = 8,          # 8 is minimum required length by Security Hub IAM.7 contorl
   RequireUppercaseCharacters = True,  # at least one uppercase character from alphabet (A to Z)
   RequireLowercaseCharacters = True,  # at least one lowercase character from alphabet (a to z)
)
```

## Lab Walkthrough ##
The lab consists of three parts. We will first verify the we have the default password policy configured in the AWS account. Next, we will deploy the CloudFormation template to make the account password policy configuration stronger. Finally, we will test the password policy change to verify that we cannot create an IAM user with a weak password.<br/>

### Part 1 - Check current password policy: ###
1. Login to AWS console with the credentials provided for you.<br>
2. Open the [AWS IAM console](https://console.aws.amazon.com/iam/).<br>
3. Under **Access management**, click on [Account settings](https://console.aws.amazon.com/iam/home#/account_settings).<br>
4. Observe that this AWS account uses the following default password policy.<br>
![Screenshot](https://github.com/aws-samples/devlab-iam-password-policy/blob/main/images/Default_Password_Policy.jpg)<br/>
Note: If you see a custom password policy defined, click on the **Delete** button and acknowledge the confirmation message. 

### Part 2 - Update password policy configuration: ###
5. Open the [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation/).<br>
6. Click on **Create resources** *With new resources (standard)*.<br>
7. Under *Create stack*, click on **Upload a template file**.<br>
8. Download the [CloudFormation template file](devlab-iam-password-policy-change.yml) to your local disk.<br>
9. Click on **Choose file** and select the file "devlab-iam-password-policy-change.yml" downloaded in previous step.<br>
10. Wait till file is uploaded, and click **Next**.<br>
11. Under *Specify stack details*, set the **Stack name** to "devlab-iam-password-policy" and click **Next**.<br>
12. Under *Configure stack options*, click **Next**.<br>
13. Under *Review devlab-iam-password-policy*, check **I acknowledge that AWS CloudFormation might create IAM resources** and click on **Create stack*.<br>
14. The CloudFormation Stack status should be **CREATE_IN_PROGRESS**.<br>
![Screenshot](https://github.com/aws-samples/devlab-iam-password-policy/blob/main/images/Stack_Create_In_Process.jpg)<br/>
15. Refresh after 1 minute to see the status changed to **CREATE_COMPLETE**.<br>
![Screenshot](https://github.com/aws-samples/devlab-iam-password-policy/blob/main/images/Stack_Create_Complete.jpg)<br/>
16. Go back to the [AWS IAM console](https://console.aws.amazon.com/iam/).<br>
17. Under **Access management**, click on [Account settings](https://console.aws.amazon.com/iam/home#/account_settings).<br>
18. Observe that this AWS account uses now custom password policy with strong configuration.<br>
![Screenshot](https://github.com/aws-samples/devlab-iam-password-policy/blob/main/images/Strong_Password_Policy.jpg)<br/>

### Part 3 - Test password policy configuration: ### 
19. Go to the [AWS IAM console](https://console.aws.amazon.com/iam/).<br>
20. Under **Access management**, click on [Users](https://console.aws.amazon.com/iam/home#/users).<br>
21. Click on **Add users**.<br>
22. Under *Set user details*, set the **User name** to "devlab-test-user".<br>
23. Under *Select AWS access type*, choose **Password - AWS Management Console access** and change *Console password* to **Custom password**.<br>
24. Try to create this user with a weak password and observe that you get a warning message and the **Next: Permissions** button is disable.
![Screenshot](https://github.com/aws-samples/devlab-iam-password-policy/blob/main/images/Create_User_Password.jpg)<br/>

## Lab Bonus ##
If you have your [AWS Organizations](https://aws.amazon.com/organizations/) organization already created, consider implementing a [Service Control Policy (SCP)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) to disallow member accounts from resetting their IAM Password Policies. This step will not be covered in this lab, but is an additional governance mechanism you can apply across multiple accounts or the entire AWS Organization. Firstly, use [AWS CloudFormation StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html) to deploy the same AWS CloudFormation template we used in this lab across all your organization's member accounts. Once the IAM Password Policy of member accounts has been updated according to your organization standard, you can attach the SCP below using AWS Organizations to deny further updates to the IAM Password Policy by member accounts.<br/>
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUpdateAccountPasswordPolicy",
      "Effect": "Deny",
      "Action": "iam:UpdateAccountPasswordPolicy",
      "Resource": "*"
    }
  ]
}
```

## Lab Cleanup ##
This lab applies a custom IAM Password Policy to an AWS account using an AWS CloudFormation template. Deletion of the CloudFormation stack has no impact on the current password policy. To reset the account password policy to its default configuration, you can deploy the [CloudFormation reset template](devlab-iam-password-policy-reset.yml) provided to delete the custom password policy created.<br>

## Lab Survey ##
We would really appreciate if you can complete the lab survey (link below). This will help us improve your lab experience next time :) 

![qr_code](/qr/cdk-apprunner-ecs.png)  

[Survey Link](https://eventbox.dev/survey/YO8E4WA)
