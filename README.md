# python-s3-openshift

To set up your Python script running as a CronJob in OpenShift to access an S3 bucket, you need to configure AWS credentials securely using OpenShift's Cloud Credential Operator (CCO) and IAM Role-based access. Below are the step-by-step instructions:

1. Prerequisites
An OpenShift cluster with the Cloud Credential Operator (CCO) enabled.
An S3 bucket in AWS.
AWS IAM permissions to create roles, policies, and an OIDC provider.
The Python script containerized and deployed in OpenShift as a CronJob.
2. Create an AWS IAM Role

You need to create an IAM role that allows your Python script to access the S3 bucket.

Step 2.1: Create a Trust Policy

This trust policy allows OpenShift to assume the IAM role using OIDC.

json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER_URL>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER_URL>:sub": "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT_NAME>"
        }
      }
    }
  ]
}

Replace <AWS_ACCOUNT_ID> with your AWS account ID.
Replace <OIDC_PROVIDER_URL> with the OIDC provider URL for your OpenShift cluster (e.g., openshift-cluster-id.region.eks.amazonaws.com).
Replace <NAMESPACE> with the namespace where your CronJob will run.
Replace <SERVICE_ACCOUNT_NAME> with the name of the service account (e.g., s3-access-sa).
Step 2.2: Attach an IAM Policy

Create an IAM policy that grants access to the S3 bucket. For example:

json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<BUCKET_NAME>",
        "arn:aws:s3:::<BUCKET_NAME>/*"
      ]
    }
  ]
}

Replace <BUCKET_NAME> with the name of your S3 bucket.

Attach this policy to the IAM role created in Step 2.1.

3. Configure the OIDC Provider in AWS

If not already configured, set up an OIDC provider in AWS for your OpenShift cluster.

Step 3.1: Retrieve the OIDC Provider URL
Run the following command in your OpenShift cluster:
bash
oc get authentication.config.openshift.io/cluster -o jsonpath='{.spec.serviceAccountIssuer}'

This will return the OIDC provider URL.
Step 3.2: Create the OIDC Provider in AWS
Go to the IAM Management Console.
Navigate to Identity Providers and create a new provider:
Provider Type: OpenID Connect.
Provider URL: Use the URL retrieved in Step 3.1.
Audience: Use sts.amazonaws.com.
4. Create a Service Account in OpenShift

Create a service account that will be used by your CronJob to assume the IAM role.

yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access-sa
  namespace: <NAMESPACE>


Replace <NAMESPACE> with the namespace where your CronJob will run.

5. Create a CredentialsRequest in OpenShift

The CredentialsRequest resource links the service account to the IAM role.

yaml
apiVersion: cloudcredential.openshift.io/v1
kind: CredentialsRequest
metadata:
  name: s3-access
  namespace: <NAMESPACE>
spec:
  providerSpec:
    apiVersion: cloudcredential.openshift.io/v1
    kind: AWSProviderSpec
    statementEntries:
    - action:
      - s3:GetObject
      - s3:PutObject
      - s3:ListBucket
      effect: Allow
      resource:
      - "arn:aws:s3:::<BUCKET_NAME>"
      - "arn:aws:s3:::<BUCKET_NAME>/*"
  secretRef:
    name: s3-access-secret
    namespace: <NAMESPACE>
  serviceAccountNames:
  - s3-access-sa

Replace <NAMESPACE> with your namespace.
Replace <BUCKET_NAME> with your S3 bucket name.
6. Deploy the CronJob

Modify your CronJob to use the service account and mount the credentials.

Step 6.1: Example CronJob YAML
yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: s3-cronjob
  namespace: <NAMESPACE>
spec:
  schedule: "*/5 * * * *" # Adjust the schedule as needed
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: s3-access-sa
          containers:
          - name: s3-script
            image: <YOUR_IMAGE>
            command: ["python", "/path/to/your/script.py"]
            env:
            - name: AWS_REGION
              value: "<AWS_REGION>"
          restartPolicy: OnFailure

Replace <NAMESPACE> with your namespace.
Replace <YOUR_IMAGE> with the container image for your Python script.
Replace <AWS_REGION> with the AWS region of your S3 bucket.
7. Verify the Setup
Deploy the service account, CredentialsRequest, and CronJob.
Check that the pod created by the CronJob can access the S3 bucket.

You can verify this by adding logging to your Python script or using aws s3 commands in your container to list bucket contents.

How it Works
The service account (s3-access-sa) is linked to the IAM role via the OIDC provider.
When the CronJob runs, the pod assumes the IAM role and uses temporary credentials to access the S3 bucket.
The credentials are securely managed by OpenShift and AWS, so no long-lived secrets are exposed.

This setup ensures secure and seamless access to S3 for your Python script running in OpenShift.
