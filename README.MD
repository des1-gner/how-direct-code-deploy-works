From my investigation, the AgentCore Control Plane validates S3 access against the calling principal (the IAM role making the UpdateAgentRuntime or CreateAgentRuntime API call) before accepting the request (specific to direct code deployment method), meaning that access control can only be enforced at the calling principal level, rather than at the individual runtime level.

That said, the following options are available and can be used individually or combined to achieve a greater degree of access control. Note that each of these examples can be altered to use either an Allow or Deny effect depending on your specific requirements:

Option 1 - S3 Bucket Policy scoped to specific principals:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenySpecificRoleOnly",
            "Effect": "Deny",
            "Principal": {
                "AWS": "arn:aws:iam::<your-account-id>:role/<your-iam-role>"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<your-bucket-name-and-path>/deployment.zip"
        }
    ]
}

Option 2 - IAM Role Policy denying access to specific S3 objects (NOTE: This is NOT the Runtime Role):

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowAdmin",
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        },
        {
            "Sid": "DenySpecificS3Object",
            "Effect": "Deny",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::<your-bucket-name-and-path>/deployment.zip",
                "arn:aws:s3:::<your-bucket-name-and-path>"
            ]
        }
    ]
}

NOTE: This is a simplified example, please scope the Allow statement to your own least-privilege requirements.

Option 3 - Scoping UpdateAgentRuntime to specific Runtime ARNs (again NOT the Runtime Role):

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUpdateSpecificRuntime",
            "Effect": "Deny",
            "Action": "bedrock-agentcore:UpdateAgentRuntime",
            "Resource": "arn:aws:bedrock-agentcore:us-east-1:<your-account-id>:runtime/<your-agentcore-runtime-arn>"
        }
    ]
}

Additionally, from my testing, once a CreateAgentRuntime or UpdateAgentRuntime call has been successfully made, S3 access is no longer required for the runtime to continue functioning. This is because the AgentCore Control Plane retrieves the deployment artifact from S3 only at the time of creation or update, after which the code is bundled and stored within the service infrastructure. This means you can safely revoke S3 access after each deployment, and only re-grant it when you need to update the runtime with a new deployment zip.
