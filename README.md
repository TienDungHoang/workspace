# Coworking Space Service Extension
The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

For this project, you are a DevOps engineer who will be collaborating with a team that is building an API for business analysts. The API provides business analysts basic analytics data on user activity in the service. The application they provide you functions as expected locally and you are expected to help build a pipeline to deploy it in Kubernetes.

## Getting Started

### Dependencies
#### Local Environment
1. Python Environment - run Python 3.6+ applications and install Python dependencies via `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster
4. `helm` - apply Helm Charts to a Kubernetes cluster

#### Remote Resources
1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. Kubernetes Environment with AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code

### Setup
#### 1. Create an EKS Cluster
1. Check IAM Permission: 
```bash
aws sts get-caller-identity
aws configure
aws configure set aws_session_token
```

2. Create an EKS Cluster: 
```bash
eksctl create cluster --name my-cluster --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --nodes 1 --nodes-min 1 --nodes-max 2
```
3. Update Kubeconfig: 
```bash
aws eks --region us-east-1 update-kubeconfig --name my-cluster
```
Verify and copy the context name
```bash
kubectl config current-context
```

4. Delete EKS Cluster after use: 
```bash
eksctl delete cluster --name my-cluster --region us-east-1
```

#### 2. Configure a Database
Set up a Postgres database using a Helm Chart.

1. Install Helm: 
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

2. Set up Bitnami Repo:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

3. Install PostgreSQL:
```bash
helm install my-postgres bitnami/postgresql
```

4. Check the installation: 
```bash
helm list
kubectl get namespace
kubectl get pods
```

5. Apply YAML configurations
```bash
kubectl apply -f pvc.yaml
kubectl apply -f pv.yaml
kubectl apply -f postgresql-deployment.yaml
```

6. Create postgresql-service: 
```bash
kubectl apply -f postgresql-service.yaml
```

7. Setup port-forwarding: 
# List the services
```bash
kubectl get svc
```
# Set up port-forwarding to `postgresql-service`
```bash
kubectl port-forward service/postgresql-service 5433:5432 &
```

8. Run seed files: 
# Install psql on workspace: 
```bash
apt update
apt install postgresql postgresql-contrib
```

9. Create tables and populate them with data: 
```bash
export DB_PASSWORD=mypassword
export LC_ALL=C
PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433 < /workspace/db/1_create_tables.sql
PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433 < /workspace/db/2_seed_users.sql
PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433 < /workspace/db/3_seed_tokens.sql
```

10. Check the tables 
```bash
PGPASSWORD="$DB_PASSWORD" psql --host 127.0.0.1 -U myuser -d mydatabase -p 5433
select* from users;
select* from tokens;
```







This should set up a Postgre deployment at `<SERVICE_NAME>-postgresql.default.svc.cluster.local` in your Kubernetes cluster. You can verify it by running `kubectl svc`

By default, it will create a username `postgres`. The password can be retrieved with the following command:
```bash
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default <SERVICE_NAME>-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

echo $POSTGRES_PASSWORD
```

<sup><sub>* The instructions are adapted from [Bitnami's PostgreSQL Helm Chart](https://artifacthub.io/packages/helm/bitnami/postgresql).</sub></sup>

3. Test Database Connection
The database is accessible within the cluster. This means that when you will have some issues connecting to it via your local environment. You can either connect to a pod that has access to the cluster _or_ connect remotely via [`Port Forwarding`](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)

* Connecting Via Port Forwarding
```bash
kubectl port-forward --namespace default svc/<SERVICE_NAME>-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
```

* Connecting Via a Pod
```bash
kubectl exec -it <POD_NAME> bash
PGPASSWORD="<PASSWORD HERE>" psql postgres://postgres@<SERVICE_NAME>:5432/postgres -c <COMMAND_HERE>
```


### 2. Running the Analytics Application Locally
In the `analytics/` directory:

1. Install dependencies
```bash
# Update the local package index with the latest packages from the repositories
apt update

# Install a couple of packages to successfully install postgresql server locally
apt install build-essential libpq-dev

# Update python modules to successfully build the required modules
pip install --upgrade pip setuptools wheel

# Install all modules listed in requirement.txt
pip install -r /workspace/analytics/requirements.txt
```

2. Run the application (see below regarding environment variables)
```bash
export DB_USERNAME=myuser
export DB_PASSWORD=${POSTGRES_PASSWORD}
export DB_HOST=127.0.0.1
export DB_PORT=5433
export DB_NAME=mydatabase
python /workspace/analytics/app.py
```

There are multiple ways to set environment variables in a command. They can be set per session by running `export KEY=VAL` in the command line or they can be prepended into your command.

* `DB_USERNAME`
* `DB_PASSWORD`
* `DB_HOST` (defaults to `127.0.0.1`)
* `DB_PORT` (defaults to `5432`)
* `DB_NAME` (defaults to `postgres`)

If we set the environment variables by prepending them, it would look like the following:
```bash
DB_USERNAME=username_here DB_PASSWORD=password_here python app.py
```
The benefit here is that it's explicitly set. However, note that the `DB_PASSWORD` value is now recorded in the session's history in plaintext. There are several ways to work around this including setting environment variables in a file and sourcing them in a terminal session.

3. Verifying The Application
* Generate report for check-ins grouped by dates
`curl <BASE_URL>/api/reports/daily_usage`

* Generate report for check-ins grouped by users
`curl <BASE_URL>/api/reports/user_visits`



### 3. Deploy the analytics application: 
1. Dockerize the Application: 
# Update docker version
```bash
apt update
apt install docker-ce docker-ce-cli containerd.io
```
# Build and run a docker image from Dockerfile
```bash
docker build -t coworking .
```

2. Create ECR Repository and push docker image to repository
![Alt text](https://vscode-remote%2B7cslhx2b6p-002eprod-002eudacity-002dstudent-002dworkspaces-002ecom.vscode-resource.vscode-cdn.net/workspace/Project%20Screenshots/ECR%20Repository.png?version%3D1713895726110)


3. Push to Github
```bash
git config --global user.email "DungHoang1989@gmail.com"
git config --global user.name "TienDungHoang"
```




## Project Instructions
1. Set up a Postgres database with a Helm Chart
2. Create a `Dockerfile` for the Python application. Use a base image that is Python-based.
3. Write a simple build pipeline with AWS CodeBuild to build and push a Docker image into AWS ECR
4. Create a service and deployment using Kubernetes configuration files to deploy the application
5. Check AWS CloudWatch for application logs

### Deliverables
1. `Dockerfile`
2. Screenshot of AWS CodeBuild pipeline
3. Screenshot of AWS ECR repository for the application's repository
4. Screenshot of `kubectl get svc`
5. Screenshot of `kubectl get pods`
6. Screenshot of `kubectl describe svc <DATABASE_SERVICE_NAME>`
7. Screenshot of `kubectl describe deployment <SERVICE_NAME>`
8. All Kubernetes config files used for deployment (ie YAML files)
9. Screenshot of AWS CloudWatch logs for the application
10. `README.md` file in your solution that serves as documentation for your user to detail how your deployment process works and how the user can deploy changes. The details should not simply rehash what you have done on a step by step basis. Instead, it should help an experienced software developer understand the technologies and tools in the build and deploy process as well as provide them insight into how they would release new builds.


### Stand Out Suggestions
Please provide up to 3 sentences for each suggestion. Additional content in your submission from the standout suggestions do _not_ impact the length of your total submission.
1. Specify reasonable Memory and CPU allocation in the Kubernetes deployment configuration
2. In your README, specify what AWS instance type would be best used for the application? Why?
3. In your README, provide your thoughts on how we can save on costs?

### Best Practices
* Dockerfile uses an appropriate base image for the application being deployed. Complex commands in the Dockerfile include a comment describing what it is doing.
* The Docker images use semantic versioning with three numbers separated by dots, e.g. `1.2.1` and  versioning is visible in the  screenshot. See [Semantic Versioning](https://semver.org/) for more details.