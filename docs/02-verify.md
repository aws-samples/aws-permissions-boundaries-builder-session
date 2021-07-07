# Permission Boundaries: How to Truly Delegate Permissions on AWS (Verify Phase)

We are now in the Verify phase. It is time to put on the hat of the web administrator and test the access to ensure the permissions are setup correctly. You'll be logging in to the AWS account you were delegated access to and perform a number of verification tasks.

## Console Login

You should have received from another team the following information. You will need this information to access the AWS console 

* IAM users sign-in link
* IAM user name
* IAM user password
* Resource restriction identifier
* Permission boundary name

## Requirements

The only requirement is to verify you can complete the following tasks. 

The web admins should only have access to the following resources:

1. IAM policies and roles created by the web admins
2. Lambda functions created by the web admins

The web admins should not be able to impact any resources in the account that they do not own including users, groups, roles, S3 buckets, EC2 instances, etc.

The following steps should be taken to validate that the delegation was done properly. Verify that you are able to create an IAM policy, create an IAM role with that policy attached and then create a Lambda function and pass that role to it.

## Task 1 - Create a customer managed IAM policy
	
* The first step is to create a customer managed IAM policy. This will define the permissions of the role that you will pass to a Lambda function. Since the function will be working with S3 and since the point of this is to show how permission boundaries work, use the following policy which grants basic Lambda logging permissions and S3 full access. 

> Keep in mind the resource restrictions put in place which will require you to use a certain name for the policy.  What is the resource restriction identifier that was given to you?

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "s3:*"
            ],
            "Resource": "*"
        }
    ]
}
```

## Task 2 - Create an IAM role

* Next you will create an IAM Role. Choose Lambda as the service for this role. Attach the policy you just created and the permission boundary (which will most likely be named:  **identity-ex-permissionboundary-ares-lambda**)

> Keep in mind the resource restrictions put in place which will require you to use a certain name for the role.  What is the resource restriction identifier that was given to you?

## Task 3 - Create a Lambda function

* Finally you will create a **Node.js 8.10** Lambda function using the code below and attach the IAM role you just created to it. You will need to replace `"ELB_ACCESS_LOGS_BUCKET_NAME"` with bucket from your account that begins with `"identity-ex-ares*"`. 


> Keep in mind the resource restrictions put in place which will require you to use a certain name for the role.  What is the resource restriction identifier that was given to you?


```
const AWS = require('aws-sdk');
const s3 = new AWS.S3();

exports.handler = async (event) => {
  console.log('Loading function');
  const allKeys = [];
  await getKeys({ Bucket: 'ELB_ACCESS_LOGS_BUCKET_NAME' }, allKeys);
  return allKeys;
};

async function getKeys(params, keys){
  const response = await s3.listObjectsV2(params).promise();
  response.Contents.forEach(obj => keys.push(obj.Key));

  if (response.IsTruncated) {
    const newParams = Object.assign({}, params);
    newParams.ContinuationToken = response.NextContinuationToken;
    await getKeys(newParams, keys); 
  }
}
```

* Test the Lambda function and make sure it is generating logs in CloudWatch logs and that it is able to list the ELB logs in the ELB access logs bucket the object in S3. In order to test you will need to create a test event. The parameters of the test do not matter.

* You can perform an additional test after changing the `"ELB_ACCESS_LOGS_BUCKET_NAME"` with the bucket that begins with `"identity-ex-rover*"`. In that case, the Lambda function won't have the permission to list files, as we specified the ares bucket only in the permission boundaries.

## Cleanup

In order to prevent charges to your account we recommend cleaning up the infrastructure that was created. Expand one of the following dropdowns and follow the instructions:

??? note "AWS Sponsored Event"
    No cleanup required! The responsibility falls to AWS.

??? note "Individual"

    You will need to manually delete some resources before you delete the CloudFormation stacks so please do the following steps in order.

    1.	Delete the Ares S3 bucket.
        * Go to the <a href="https://s3.console.aws.amazon.com/s3/home" target="_blank">Amazon S3</a> console.
        * Click on the bucket named **identity-ex-ares-app**
        * Click **Delete**
        * Enter the bucket name again to confirm and click **Delete**.

    2.	Delete the CloudFormation stack (**Identity-PB-Builder-Session**).
        * Go to the <a href="https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks?filter=active" target="_blank">AWS CloudFormation</a> console.
        * Select the appropriate stack.
        * Select **Action**.
        * Click **Delete Stack**.

## Summary

Congratulations, you've completed the Permission Boundaries Builder session!  Hopefully by going through this session you have a better idea of what permission boundaries are, where they can be used, and are starting to think about where you can apply them in your environments.
