## Environment Setup

To setup your environment please expand one of the following dropdown sections (depending on how you're doing this builder session) and follow the instructions:

??? info  "Click here if you're at an *AWS event* where the *Event Engine* is being used"

    <p style="font-size:20px;">
      **Step 1** : Open the AWS Console
    </p>

	1. Navigate to the <a href="https://dashboard.eventengine.run" target="_blank">Event Engine dashboard</a>
	2. Enter your **team hash** code.
	3. Click **AWS Console**.  The CloudFormation template for this round has already been prerun.

??? info "Click here if you're running this individually in your own AWS Account"

    Launch the CloudFormation stack below to setup the Permission Boundary environment:

    Region| Deploy
    ------|-----
    US West 2 (Oregon) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=Identity-PB-Builder-Session&templateURL=https://s3-us-west-2.amazonaws.com/sa-security-specialist-workshops-us-west-2/builder-sessions/permissionboundary/permission-boundaries-env.yml" target="_blank">![Deploy in us-west-2](./images/deploy-to-aws.png)</a>

    1. Click the **Deploy to AWS** button above.  This will automatically take you to the console to run the template.  

    2. Click **Next** on the **Specify Template**, **Specify Details, and **Options** sections.

    3. Finally, acknowledge that the template will create IAM roles under **Capabilities and click **Create**.

    This will bring you back to the CloudFormation console. You can refresh the page to see the stack starting to create. Before moving on, make sure the stack is in a **CREATE_COMPLETE**.


> All resources are located in the **us-west-2** region.

## Task 1 - Create a permission boundary for Lambda Functions

**ACTION**: Create a new IAM policy that will act as the permission boundary for the web admins. Name the policy **`identity-ex-permissionboundary-ares-lambda`**

>  **Hint**: <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html" target="_blank">Friendly Names and Paths
</a>. Replace the ACCOUNT_ID with your account ID and the **'< >'** with a friendly name that can be used as a resource restriction.  

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CreateLogGroup",
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:us-west-2:ACCOUNT_ID:*"
        },
        {
            "Sid": "CreateLogStreamandEvents",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:us-west-2:ACCOUNT_ID:log-group:/aws/lambda/identity-ex-<>:*"
        },
        {
            "Sid": "AllowedS3GetObject",
            "Effect": "Allow",
            "Action": [
                "s3:List*"
            ],
            "Resource": "arn:aws:s3:::identity-ex-ares-*"
        }
    ]
}
```

## Task 2 - Create a permission policy for the Web Admin

**ACTION**: Create the permission policy that will be attached to the **webadmin** AWS IAM user. Name the new policy **`identity-ex-webadmin-permissionpolicy`**.

>  **Hint**: <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html" target="_blank">Friendly Names and Paths
</a>. Replace the ACCOUNT_ID with your account ID and the **'< >'** with a friendly name that can be used as a resource restriction.

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CreateCustomerManagedPolicies",
            "Effect": "Allow",
            "Action": [
                "iam:CreatePolicy",
                "iam:DeletePolicy",
                "iam:CreatePolicyVersion",
                "iam:DeletePolicyVersion",
                "iam:SetDefaultPolicyVersion"
            ],
            "Resource": "arn:aws:iam::ACCOUNT_ID:policy/identity-ex-<>"
        },
        {
              "Sid": "RoleandPolicyActionswithnoPermissionBoundarySupport",
            "Effect": "Allow",
            "Action": [
                    "iam:UpdateRole",
                    "iam:DeleteRole"
            ],
            "Resource": [
                "arn:aws:iam::ACCOUNT_ID:role/identity-ex-<>"
            ]
        },
        {
            "Sid": "CreateRoles",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy"
            ],
            "Resource": [
                "arn:aws:iam::ACCOUNT_ID:role/identity-ex-<>"
            ],
            "Condition": {"StringEquals":
                {"iam:PermissionsBoundary": "arn:aws:iam::ACCOUNT_ID:policy/identity-ex-permissionboundary-ares-lambda"}
            }
        },
        {
            "Sid": "LambdaFullAccesswithResourceRestrictions",
            "Effect": "Allow",
            "Action": "lambda:*",
            "Resource": "arn:aws:lambda:us-west-2:ACCOUNT_ID:function:identity-ex-<>"
        },
        {
            "Sid": "PassRoletoLambda",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::ACCOUNT_ID:role/identity-ex-<>",
            "Condition": {
                "StringLikeIfExists": {
                    "iam:PassedToService": "lambda.amazonaws.com"
                }
            }
        },
        {
            "Sid": "AdditionalPermissionsforLambda",
            "Effect": "Allow",
            "Action":   ["kms:ListAliases", "logs:Describe*", "logs:ListTagsLogGroup", "logs:FilterLogEvents", "logs:GetLogEvents"],
            "Resource": "*"
        },
        {
            "Sid": "DenyPermissionBoundaryandPolicyDeleteModify",
            "Effect": "Deny",
            "Action": [
                "iam:CreatePolicyVersion",
                "iam:DeletePolicy",
                "iam:DeletePolicyVersion",
                "iam:SetDefaultPolicyVersion"
            ],
            "Resource": [
                "arn:aws:iam::ACCOUNT_ID:policy/identity-ex-permissionboundary-ares-lambda",
                "arn:aws:iam::ACCOUNT_ID:policy/identity-ex-webadmin-permissionpolicy"
            ]
        },
        {
            "Sid": "DenyRolePermissionBoundaryDelete",
            "Effect": "Deny",
            "Action": "iam:DeleteRolePermissionsBoundary",
            "Resource": "*"
        }
    ]
}
```

## Task 3 - Create the Web Admin user

**ACTION**:

* Create an IAM User and name it `webadmin`. The user will need console access so give it a password.
* Attach the **identity-ex-webadmin-permissionpolicy**, **IAMReadOnlyAccess** & **AWSLambda_ReadOnlyAccess** policies to the IAM user.

When you are done the **webadmin** user should have three policies attached: identity-ex-webadmin-permissionpolicy, IAMReadOnlyAccess & AWSLambda_ReadOnlyAccess.

## Task 4 - Gather info needed for the Verify phase

**ACTION**: Now that you have setup the IAM user for the web admins, it's time to pass this information on to the next team who will work through the **VERIFY** tasks. You need to gather some details about your IAM user and then hand this info to the next team.

Copy the **IAM users sign-in link**, the IAM user name (if you used a name other then **webadmin**) and the password you used. You will also need the resource restriction that you used in your policies and the name you used for the permission policy and permission boundary (if you used names other than the ones recommended above)

Here are all of the details you need to pass to another team:

* IAM users sign-in link:   
* IAM user name:    
* IAM user password:    
* Resource restriction identifier:  
* Permission boundary name: (recommended name: **identity-ex-permissionboundary-ares-lambda**)
* Permission policy: (recommended name: **identity-ex-webadmin-permissionpolicy**)

Enter this information into the **VERIFY** phase form and exchange forms with another team so you both can work through the tasks.

You can now move on to the **Verify** phase!
