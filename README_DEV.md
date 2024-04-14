## Deployment guide

### Preconditions
1. Install eksctl, aws cli and kubectl.
2. Push the code to Github repo.

### Build CI process
1. Create Dockerfile
- The application is Python based. So, please use base image with Python installed. In this case I am using `python:3.9`
- Specific the working directory. eg: /app
- Copy file requirements.txt from source code to working directory
- Install python dependecies
- Copy all python source code to the working directory
- Run application with `python` command.

2. Create buildspec.yaml file
- Get AWS password and login docker to use ECR as image repository.
`aws ecr get-login-password | docker login --username AWS --password-stdin $API_REPOSITORY_URI`
- Run docker build to build source code to a image.
`docker build -t $API_REPOSITORY_URI:latest -f Dockerfile .`
`docker tag $API_REPOSITORY_URI:latest $API_REPOSITORY_URI:$IMAGE_TAG`
- Push built docker image to ECR.
`docker push $API_REPOSITORY_URI:$IMAGE_TAG`

3. Create ECR repository
aws ecr create-repository  --repository-name analytics

4. Create Codebuild project
- Source: Select Github and choose the repo which contain the analytics application
- Buildspec enter `analytics/buildspec.yaml`
- You can enable auto build when has new code changes on Github repository. Or you can click on button `Start build` to run manually.
- After building success, you can see the image on ECR repository


### Deployment steps
1. Create EKS cluster
`eksctl create cluster --name {{CLUSTER_NAME}} --region us-east-1 --nodegroup-name {{NODE_GROUP_NAME}} --node-type t3.small --nodes 1 --nodes-min 1 --nodes-max 2`

Connect EKS with local kubectl `aws eks update-kubeconfig --region us-east-1 --name {{CLUSTER_NAME}}`

After done, delete cluster by below command
`eksctl delete cluster --name {{CLUSTER_NAME}} --region us-east-1`

2. Add cloudwatch to the EKS cluster
- Add new policy `CloudWatchAgentServerPolicy` to EKS Node group role
`aws iam attach-role-policy --role-name {{EKS_NODE_GROUP_ROLE_NAME}} --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy`

- Add addon cloudwatch to EKS
`aws eks create-addon --cluster-name {{CLUSTER_NAME}} --addon-name amazon-cloudwatch-observability`

3. Deploy database
Go to the db folder inside repository. Run below command to deploy postgres database
`kubectl apply -f pvc.yaml`
`kubectl apply -f pv.yaml`
Review and update the database info inside file `postgresql-deployment.yaml` if needed: POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
`kubectl apply -f postgresql-deployment.yaml`
`kubectl apply -f postgresql-service.yaml`

Connect to database and run below script to create test data
`1_create_tables.sql`
`2_seed_users.sql`
`3_seed_tokens.sql`

3. Create secret for database
Go to the folder deployment
Update the password value to match with POSTGRES_PASSWORD inside file `dbsecret.yaml`
Run command to create secret: `kubectl apply -f dbsecret.yaml`

4. Deploy coworking application
Update the value for below variables: DB_HOST, DB_PORT, DB_USERNAME, DB_PASSWORD, DB_NAME inside file `coworking.yaml`.
Anytime you have new version of code, please change the version in this section `image: 402846102310.dkr.ecr.us-east-1.amazonaws.com/analytics:15`

Run below command to deploy application
`kubectl apply -f coworking.yaml`

### Verify
Get all services that deployed to cluster
`kubectl get svc`

Get all pods
`kubectl get pods`

You also able to check application logs from CloudWatch


## Stand-Out Suggestions
1. Specify reasonable Memory and CPU allocation in the Kubernetes deployment configuration
The application is very small. So, I think we can start with 2 CPU and 2 GB RAM for development environment.
Based on the workload on PROD environment, we can consider larger CPU and RAM.

2. In your README, specify what AWS instance type would be best used for the application? Why?
For development we can use t3a.small. t3a instance is using ADM chip and it is cheaper than t3 with the same CPU and RAM
For PROD environment, we can use larger t3a instance.

3. In your README, provide your thoughts on how we can save on costs?
For saving cost, we should consider below items:
- Select appropriate instance type based on the actual workload.
- Implement autoscaling for the EKS cluster. If the current node are using more than 70% of CPU, system will add more node.
- For CloudWatch, setup log retention to 15 days. It is enough for investigating issues.
