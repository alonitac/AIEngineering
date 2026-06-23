# AWS Bedrock

Amazon Bedrock is an AWS service that gives you access to LLMs from leading model providers - Anthropic, OpenAI, Meta, and others - through a single API.

In this course we have a **shared AWS account**. Each student has their own IAM user with credentials. You will use those credentials to call Bedrock models from your code.


## Install the AWS CLI (Ubuntu)

The AWS CLI lets you talk to AWS services from the command line.

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify the installation:

```bash
aws --version
```


## Configure Your AWS Credentials

To use AWS CLI, you need to generate and configure your credentials. Follow these steps:

1. Sign in to the AWS console at [https://console.aws.amazon.com](https://console.aws.amazon.com).
2. Click your username in the top-right corner - **Security credentials**.
3. Scroll down to **Access keys** - click **Create access key**.
4. Select **Command Line Interface (CLI)** as the use case - check the confirmation box - click **Next**.
5. Click **Create access key**.
6. **Copy both values now** - the Secret Access Key is shown only once and cannot be retrieved later.

In your Ubuntu terminal, run:

```bash
aws configure
```

You will be prompted for four values:

```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: blablablablabla
Default region name [None]: us-east-1
Default output format [None]: json
```

This writes your credentials to `~/.aws/credentials` and your config to `~/.aws/config`. When using AWS CLI or AWS SDK through Python, the credentials are automatically picked up from these files - **NO NEED TO COPY THESE VALUES ANYWHERE, EVER, NEVER, NOWHATEVER YOU DO**.

## Using Bedrock with LangChain

Install the Required Packages

```bash
pip install langchain-aws
```


Simple usage:

```python
from langchain.chat_models import init_chat_model

model = init_chat_model(
    "bedrock/anthropic.claude-3-haiku-20240307-v1:0",
    region_name="us-east-1",
)

response = model.invoke("Hello!")
print(response.content)
```

## Allowed models 

- `anthropic.claude-3-haiku-20240307-v1:0`,
- `amazon.nova-micro-v1:0`,
- `amazon.nova-lite-v1:0`,
- `openai.gpt-oss-20b-1:0`,
- `meta.llama3-1-8b-instruct-v1:0`,
- `mistral.mistral-7b-instruct-v0:2`

# Exercises 

### :pencil2: Use Bedrock in your PolyAI Agent

Do.