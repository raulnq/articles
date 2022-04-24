## How to assume an AWS IAM Role from an EKS Pod?


Usually, when we had to use a service from an external provider, we injected the service credential (connection string, secret, token, etc.) in the application (from a Key Vault, environment variable, app settings, etc.). But in AWS, there is the concept of [IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) that could make life much easier (and safer).

> An IAM role is similar to an IAM user, in that it is an AWS identity with permission policies that determine what the identity can and cannot do in AWS. However, instead of being uniquely associated with one person, a role is intended to be assumable by anyone who needs it.

From the text above, we can tell EKS to run a Pod using a specific IAM Role and control the access needed by the application against the AWS services. As a first step, we are going to create an IAM Role with the OpenID Connect provider URL of your EKS Cluster:

![oidc-blur.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1650740012911/7SswmnqdR.png)

Now we can create the AWS Role (AIM-> Roles -> Create Role). Choose as trusted entity type the ```Web Identity``` option:

![role-step-1-blur.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1650746624407/680voega-.png)

If your Identity Provider is not on the list, create a new one following these [instructions](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html#manage-oidc-provider-console). Time to add the permission for the IAM Role:

![role-step-2.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1650746899499/Xr7KIZp3y.PNG)

Give a name to the AIM Role and save it.

![role-step-3-blur.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1650747400683/SR6nJh0J3.png)

Go to the IAM Role recently created and change the thrust policy to:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::[AWS_ACCOUNT_ID]:oidc-provider/oidc.eks.us-east-2.amazonaws.com/id/[OIDC_ID]"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringLike": {
					"oidc.eks.us-east-2.amazonaws.com/id/[OIDC_ID]:aud": "system:serviceaccount:demo-namespace:*"
				}
			}
		}
	]
}
```

The condition used will help us assign the AIM Role to every Service Account in a specific Namespace. If this condition doesn't fit your requirements feel free to change it. Copy the AIM Role's ARN because next, we are going to create the Service Account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: demo-sa
 namespace: demo-namespace
 annotations:
   eks.amazonaws.com/role-arn:  arn:aws:iam::[AWS_ACCOUNT_ID]:role/eks-demo-role
``` 
And to asssign it to a Pod (thru a Deployment):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      serviceAccountName: demo-sa
      containers:
        - name: api-deployment-container
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Development
          image: [AWS_ACCOUNT_ID].dkr.ecr.us-east-2.amazonaws.com/demo-api:3.0
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
``` 

And that's it EKS injects the AWS session credentials based on the annotation on the Service Account used by the Pod. The credentials will get exposed by ```AWS_ROLE_ARN``` & ```AWS_WEB_IDENTITY_TOKEN_FILE``` environment variables. The AWS SDK used by your application will use these credentials to connect to the AWS services.

[Here](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) you can find more information for further exploration on it.