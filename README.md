# Using AWS services with workloads deployed at Red Hat OpenShift Service for Amazon Web Services (ROSA) 

## ROSA STS Project Preparation
The documentation:
https://docs.openshift.com/rosa/authentication/assuming-an-aws-iam-role-for-a-service-account.html#setting-up-an-aws-iam-role-a-service-account_assuming-an-aws-iam-role-for-a-service-account

Create a project: `oc new-project <PROJECT_NAMESPACE>`

Find out ROSA cluster OIDC ARN:
`
rosa describe cluster -c <CLUSTER_NAME>
`

Create `trust-policy.json`:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "<OIDC_ARN>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "<OIDC_NAME>:sub": "system:serviceaccount:<PROJECT_NAMESPACE>:<SERVICE_ACCOUNT_NAME>"
                }
            }
        }
    ]
}
```

Create a role:
`aws iam create-role --role-name <ROLE_NAME> --assume-role-policy-document file://trust-policy.json`

Fetch role ARN in a format like `arn:aws:iam::AWS_ACCOUNT_NUMBER:role/<ROLE_NAME>`.

Attach S3 read only policy:
`aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess --role-name <ROLE_NAME>`

Create `service-account.yaml`:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <SERVICE_ACCOUNT_NAME>
  namespace: sts-demo
  annotations:
    eks.amazonaws.com/role-arn: "<ROLE_ARN>"
```
And then do `oc create -f service-account.yaml`

Example of the deployment that reference the service account:
```
apiVersion: v1
kind: Pod
metadata:
  namespace: sts-demo
  name: awsboto3sdk
spec:
  serviceAccountName: <SERVICE_ACCOUNT>
  containers:
  - name: awsboto3sdk
    image: quay.io/rh_ee_ozimakov/awsboto3sdk:latest
    command:
    - /bin/bash
    - "-c"
    - "sleep 100000"
  terminationGracePeriodSeconds: 0
  restartPolicy: Never
```

The resulting pod shoul have the following environment variables injected:
 - `AWS_ROLE_ARN:<ROLE_ARN>`
 - `AWS_WEB_IDENTITY_TOKEN_FILE:  /var/run/secrets/eks.amazonaws.com/serviceaccount/token`

And the file above contains a web identity token. All AWS SDKs should be able to read it automatically.

## Local dev
```bash 
./mvnw quarkus:dev
curl -i http://localhost:8080/list
```

## Deploy to OpenShift
```
./mvnw package -Dquarkus.kubernetes.deploy=true
oc expose svc/s3-lister-service
```
Add the following elements to the generated DeploymentConfig:
 - Service account `serviceAccountName: <SERVICE_ACCOUNT_NAME>`
 - Environment variable `AWS_REGION: <AWS_REGION>`

Rollout the update.

Call the endpoint:
`
curl -i https://<ROUTE_URL>/list
`