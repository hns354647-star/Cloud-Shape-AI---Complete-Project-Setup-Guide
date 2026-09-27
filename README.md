import json
import boto3
from botocore.exceptions import ClientError

# 1. Initialize the Bedrock Runtime Client
bedrock_runtime = boto3.client(
    service_name="bedrock-runtime",
    region_name="us-east-1"  # Change to your AWS region
)

# 2. System Prompt defining Cloud Shape AI behavior
SYSTEM_PROMPT = """
You are an expert Cloud Security and Cost Optimization Architect powered by Cloud Shape AI. 

Analyze the provided Infrastructure as Code (IaC) template (CloudFormation or Terraform) and return a structured JSON evaluation covering:
1. Over-Provisioned Resources: Identify EC2 instances, RDS databases, or provisioning configurations exceeding optimal thresholds.
2. Estimated Monthly Cost Impact: Estimate potential savings by right-sizing resource specifications.
3. Automated Patch Generation: Provide corrected CloudFormation/Terraform code snippets to remediate issues.
4. Security & Compliance Rules: Flag public S3 buckets, overly permissive IAM policies, or unencrypted storage volumes.

Respond STRICTLY in valid JSON format using the following structure:
{
  "over_provisioned_resources": [],
  "estimated_monthly_savings_usd": 0,
  "remediation_patches": [],
  "security_compliance_issues": []
}
"""

# 3. Sample IaC Template for Testing
SAMPLE_IAC_TEMPLATE = """
Resources:
  MyOverprovisionedEC2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.2xlarge
      ImageId: ami-0c55b159cbfafe1f0
  UnencryptedBucket:
    Type: AWS::S3::Bucket
    Properties:
      AccessControl: PublicRead
"""

def run_cloud_shape_ai_test(iac_content: str):
    # Select Anthropic Claude model (e.g., Claude 3 Sonnet / Haiku)
    model_id = "anthropic.claude-3-sonnet-20240229-v1:0"

    payload = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 2000,
        "system": SYSTEM_PROMPT,
        "messages": [
            {
                "role": "user",
                "content": f"IaC Template Input:\n{iac_content}"
            }
        ]
    }

    try:
        response = bedrock_runtime.invoke_model(
            modelId=model_id,
            body=json.dumps(payload)
        )
        
        response_body = json.loads(response["body"].read())
        output_text = response_body["content"][0]["text"]
        
        print("--- Cloud Shape AI Scan Results ---")
        print(output_text)
        return output_text

    except ClientError as e:
        print(f"Error invoking Bedrock model: {e}")
        return None

if __name__ == "__main__":
    run_cloud_shape_ai_test(SAMPLE_IAC_TEMPLATE)
