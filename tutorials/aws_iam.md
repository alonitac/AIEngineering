# Identity and Access Management (IAM)

### Create IAM role with permissions over Bedrock and attach it to an EC2 instance

To call Bedrock models from an EC2 instance, the instance needs permission to do so. In AWS, you grant permissions to EC2 instances through **IAM roles** - not by hardcoding credentials in your code. You create a role, attach a policy that allows the specific Bedrock actions you need, and then attach that role to the instance. From that point, any code running on the instance can call Bedrock automatically, using the role's credentials behind the scenes.

To create a role with the right permissions, follow these steps:

1. Open the IAM console at [https://console\.aws\.amazon\.com/iam/](https://console.aws.amazon.com/iam/)\.

2. In the navigation pane, choose **Roles**, **Create role**\.

3. On the **Trusted entity type** page, choose **AWS service** and the **EC2** use case\. Choose **Next: Permissions**\.

4. On the **Attach permissions policy** page, choose **Create inline policy** (we will write a custom policy - no AWS managed policy covers only specific models)\.

5. On the **Review** page, enter a name for the role and choose **Create role**\.
6. Attach the role to your EC2 instance. 
7. Test your policy.

Let's review the created permission JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-3-haiku-20240307-v1:0",
        "arn:aws:bedrock:*::foundation-model/amazon.nova-micro-v1:0",
        "arn:aws:bedrock:*::foundation-model/amazon.nova-lite-v1:0",
        "arn:aws:bedrock:*::foundation-model/openai.gpt-oss-20b-1:0",
        "arn:aws:bedrock:*::foundation-model/meta.llama3-1-8b-instruct-v1:0",
        "arn:aws:bedrock:*::foundation-model/mistral.mistral-7b-instruct-v0:2"
      ]
    }
  ]
}
```

- `Version`: Denotes the version of the policy language being used.
- `Statement`: An array of policy statements, each defining a permission rule.
- `Effect`: Specifies whether the statement allows or denies access ("Allow" in this case).
- `Action`: Lists the actions allowed - `bedrock:InvokeModel` for synchronous calls and `bedrock:InvokeModelWithResponseStream` for streaming responses.
- `Resource`: Specifies exactly which foundation models may be invoked, using their full ARN.

This policy allows invoking only the six listed Bedrock foundation models, and nothing else.


## The Principle of Least Privilege

> [!NOTE]
> **The Principle of Least Privilege (PoLP)** is a security best practice that involves giving users and systems only the minimum permissions necessary to perform their tasks or functions, and no more.
> This helps to reduce the risk of accidental or intentional damage or data loss, and limit the potential impact of security breaches or vulnerabilities.


> [!TIP]
> **Validate your changes** using the [IAM Policy Simulator](https://policysim.aws.amazon.com/) - try invoking an allowed model and a model that is **not** on the list, and confirm the expected Allow/Deny results.



