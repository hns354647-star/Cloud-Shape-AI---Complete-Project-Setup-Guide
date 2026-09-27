import json
import boto3
from botocore.exceptions import ClientError

SYSTEM_PROMPT = """
You are Cloud Shape AI, an expert Cloud Security and Cost Optimization Architect.

Analyze the provided Infrastructure as Code (IaC) template (CloudFormation or Terraform) and evaluate it strictly for:
1. Over-Provisioned Resources: Identify EC2 instances, RDS databases, or provisioning configurations that exceed optimal cost/performance thresholds.
2. Estimated Monthly Cost Savings: Estimate potential monthly USD savings by right-sizing or updating resource specifications.
3. Automated Remediation Patches: Provide corrected IaC code snippets to fix over-provisioned or misconfigured resources.
4. Security & Compliance Rules: Flag public S3 buckets, overly permissive IAM policies, or unencrypted storage volumes.

Respond STRICTLY in valid JSON format using the following exact structure:
{
  "over_provisioned_resources": [
    {
      "resource_id": "string",
      "current_type": "string",
      "recommended_type": "string",
      "reason": "string"
    }
  ],
  "estimated_monthly_savings_usd": 0,
  "remediation_patches": [
    {
      "resource_id": "string",
      "patch_code": "string"
    }
  ],
  "security_compliance_issues": [
    {
      "resource_id": "string",
      "issue": "string",
      "severity": "HIGH | MEDIUM | LOW"
    }
  ]
}
"""

def scan_iac_template(iac_content: str, region_name: str = "us-east-1", model_id: str = "anthropic.claude-3-sonnet-20240229-v1:0"):
    """
    Involves Amazon Bedrock to analyze IaC code against cost and security rules.
    """
    try:
        bedrock_runtime = boto3.client(
            service_name="bedrock-runtime",
            region_name=region_name
        )

        payload = {
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 2500,
            "system": SYSTEM_PROMPT,
            "messages": [
                {
                    "role": "user",
                    "content": f"IaC Template Input:\n{iac_content}"
                }
            ]
        }

        response = bedrock_runtime.invoke_model(
            modelId=model_id,
            body=json.dumps(payload)
        )

        response_body = json.loads(response["body"].read())
        raw_output = response_body["content"][0]["text"]

        # Parse JSON response
        scan_results = json.loads(raw_output)
        return scan_results

    except ClientError as e:
        return {"error": f"AWS Bedrock Client Error: {str(e)}"}
    except json.JSONDecodeError:
        return {"error": "Failed to parse JSON response from Bedrock model."}
    except Exception as e:
        return {"error": f"Unexpected error: {str(e)}"}
import streamlit as st
import json
from core_engine import scan_iac_template

st.set_page_config(
    page_title="Cloud Shape AI",
    page_icon="☁️",
    layout="wide"
)

st.title("☁️ Cloud Shape AI")
st.subheader("Automated Infrastructure Cost Optimization & Security Analysis")

# Sidebar for AWS Settings
st.sidebar.header("Configuration")
aws_region = st.sidebar.text_input("AWS Region", value="us-east-1")
bedrock_model = st.sidebar.selectbox(
    "Bedrock Model",
    ["anthropic.claude-3-sonnet-20240229-v1:0", "anthropic.claude-3-haiku-20240307-v1:0"]
)

# Sample Template for Fast Demo
SAMPLE_CLOUDFORMATION = """
Resources:
  MyOverprovisionedEC2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.2xlarge
      ImageId: ami-0c55b159cbfafe1f0

  UnencryptedPublicBucket:
    Type: AWS::S3::Bucket
    Properties:
      AccessControl: PublicRead
"""

st.markdown("### Input IaC Template (CloudFormation / Terraform)")
iac_input = st.text_area("Paste template code below:", value=SAMPLE_CLOUDFORMATION, height=220)

if st.button("🚀 Analyze Infrastructure"):
    if not iac_input.strip():
        st.warning("Please provide an IaC template to scan.")
    else:
        with st.spinner("Analyzing infrastructure via Amazon Bedrock..."):
            results = scan_iac_template(
                iac_content=iac_input,
                region_name=aws_region,
                model_id=bedrock_model
            )

        if "error" in results:
            st.error(results["error"])
        else:
            st.success("Scan Completed!")

            # Top Metrics
            col1, col2 = st.columns(2)
            col1.metric("Est. Monthly Savings", f"${results.get('estimated_monthly_savings_usd', 0)}")
            col2.metric("Security Issues Found", len(results.get('security_compliance_issues', [])))

            st.divider()

            # Over-provisioned resources
            st.subheader("⚠️ Over-Provisioned Resources")
            st.json(results.get("over_provisioned_resources", []))

            # Security Issues
            st.subheader("🛡️ Security & Compliance Issues")
            st.json(results.get("security_compliance_issues", []))

            # Remediation Patches
            st.subheader("🛠️ Remediation Patches")
            st.json(results.get("remediation_patches", []))        
