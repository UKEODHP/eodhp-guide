# EO DataHub Platform Deployment

## Infrastructure

The EODHP is designed to be deployed to AWS. The infrastructure deployment is managed by the [Terraform deployment](https://github.com/EO-DataHub/eodhp-deploy-infrastucture) and [Supporting Terraform deployment](https://github.com/EO-DataHub/eodhp-deploy-supporting-infrastructure) repositories.

The Terraform repos manages multiple deployment environments using workspaces. Before deploying ensure you are in the correct workspace.

The deployment configuration variables are stored in the terraform/envs/\*/\*.tfvars files.

```bash
cd terraform
terraform workspace list
terraform workspace select dev
terraform apply -var-file envs/dev/dev.tfvars
```

### Workspace cloudfront distribution

The terraform deployment repository sets up a cloudfront distribution to manage traffic to the workspace domains.

By default, requests are directed to the elastic load balancer.
Traffic matching the URI form `/files/<workspace_bucket>` are instead directed to a workspace S3 bucket origin. This origin must be defined in terraform for each new bucket to be opened up to https access.

Requests intended for an S3 bucket should also pass through two Lambda@Edge functions.
- The first of these runs on `viewer-request`, and validates the access token against a secret key located in AWS secrets. Additionally, it stores the host of the request as `X-Original-Host`, since this contains the workspace name.
- The second lambda function runs on `origin-request`, and uses the workspace name and URI to redirect the request to an item in the S3 bucket.

In the repository, these functions are stored under lambda-functions. Any required modules will be installed by the terraform, and the scripts are redeployed to AWS as zip files when changes are made.

The result of this is that requests to `https://my-workspace.<workspace domain>/files/store-name/object/full/name.tif` with a valid token will be directed to an object located at `my-workspace/object/full/name.tif` in bucket `store-name`.

## Supporting Infrastructure Deployment

Before deploying the main EO DataHub Platform infrastructure, the supporting infrastructure must be deployed.

The supporting infrastructure repository contains Terraform configurations for creating and managing resources used across deployed environments such as:
 - The VPC within which all deployed clusters are hosted
 - Public NAT gateways
 - IAM roles
 - S3 buckets

The lifecycles of these resources are independent of the deployment environments (dev/test/prod).

For managing these resources, visit the [Supporting Infrastructure Terraform repository](https://github.com/EO-DataHub/eodhp-deploy-supporting-infrastructure).

### Deployment Steps

1. **Initialize Terraform**:
   Navigate to the Terraform directory where the configurations are stored.
   ```bash
   cd terraform
   terraform init # Initialize the Terraform environment
   ```

2. **Deploy the Infrastructure**:
    Ensure your AWS account ID and GitHub organization are specified in the vars/terraform.tfvars file. Then, apply the Terraform configuration.
    ```
    terraform apply -var-file vars/terraform.tfvars # Apply the Terraform configuration
    ```

Complete these steps before proceeding with the deployment of the main infrastructure.

## Applications

The platform components and applications are managed using the GitOps framework ArgoCD in the [ArgoCD deployment](https://github.com/EO-DataHub/eodhp-argocd-deployment).

The repo contains the configuration for multiple environments.

First ArgoCD has to be deployed to the cluster and then the ArgoCD application CRDs. Once ArgoCD applications have been applied then ArgoCD manages itself, as well as the platform applications.

```bash
kubectl apply -k apps/argocd
kubectl apply -k eodhp/envs/dev
```

### Deployment Repository Authorisation

If the ArgoCD deployment is private then you will have to generate a deployment key for the repository and connect the ArgoCD instance to the repository before it can read it. This can be done using the `argo` CLI or via the ArgoCD web UI.

```bash
argocd repo add git@github.com:yourusername/yourrepo.git --ssh-private-key-path ~/.ssh/repo_key
```

### Centralised Postgres Database

The supporting terraform repository configures an Aurora serverless v2 Postgres database for multiple applications to use.
The ArgoCD deployment installs and configures a Postgres operator which adds databases, users and schemas to the infrastructure, with credentials stored as kubernetes secrets. The intended design is that each application will have its own database, and each database will have a schema and user per environment.

The process for setting up a new application to use the database is as follows:
1. The following manifests should be created in a file named <app-name>-db.yaml in the apps/databases/envs/<env> directory of the argocd deployment. These resources need to be created in the same namespace as the postgres-operator deployment.
   1. Create a `Postgres` resource. This contains basic information on the new database and sets up a schema for the environment.
   2. Create a `PostgresUser` resource. This creates a user as an owner of the database, and creates kubernetes secrets for the user.
   3. Create a `Job`. This will set permissions of the user to restrict it to the environment's schema by running the script in `postgres-scripts.yaml`. It requires postgres admin credentials.
2. To allow an application to use the newly created database, credentials may be obtained using the kubernetes secret created by the PostgresUser. This exists in the databases namespace, so to fascilitate access a ClusterSecretStore, `database-store`, has been created that can fetch secrets from this namespace and replicate them in the app namespace. The keys for this secret can be set as convenient. The replicated secret data should then be passed into the application manfiests where appropriate. The secret makes the host, database name, user name and password all available.

## Manual Configuration

There are some manual steps required. While these will be automated as far as possible the current steps are outlined below.

### Keycloak

#### Update GitHub SSO configuration

In order for users to authenticate via GitHub, Keycloak needs access to the GitHub OAuth app secret.

##### First time setup

This step will not normally be required for restarting the dev or test clusters.

1. Log in to GitHub and navigate to the `KeyCloakAuth` OAuth App
2. Under `General`, navigate to `Client secrets` and click `Generate a new client secret`. Copy the secret
3. In AWS Secrets Manager, find the `eodhp-<env>` secret
4. Click `Retrieve secrets value`
5. Click `Edit`
6. Ensure that there is a key called `keycloak.auth.githubSecret`. Add if not present
7. Paste the secret in the values field
8. Save

#### Updating GitHub secret

1. Log in to AWS Secrets Manager and find the `eodhp-<env>` secret
2. Click `Retrieve secrets value`
3. Copy the value for the `keycloak.auth.githubSecret` entry
4. Navigate to the Keycloak admin panel at `<env>.eodatahub.org.uk/keycloak/admin` and log in
5. Select the `eodhp` realm
6. Click `Identity Providers`
7. Click `github`
8. Under `Settings`, update the Client Secret with the value from AWS
9. Save

#### Add Admin User to EODHP Realm

1. In Keycloak admin UI:
   1. Select eodhp realm
   2. Users
   3. Add user
      1. Set name as 'admin'
      2. Create
      3. In admin user credentials, set password (not temporary)

### Application Hub

#### Update OAUTH Client Secret

The application hub OAUTH_CLIENT_SECRET is generated by Keycloak when it is installed. A new secret is generated each time Keycloak is reinstalled.

The client secret is not automatically propagated to the App Hub, this must be done manually.

1. In Keycloak admin UI:
   1. Select eodhp realm
   2. Clients
   3. application-hub
   4. Credentials
   5. Copy Client secret
2. In AWS console:
   1. Secrets Manager
   2. eodhp-dev
   3. Retrieve secret value
   4. Edit
   5. Update app-hub.OAUTH_CLIENT_SECRET with new client secret
3. In cluster CLI:
   1. Ensure secret has propagated to app-hub secret:
      1. `kubectl get secrets -n proc app-hub -o yaml`
      2. Copy base64 encoded secret OAUTH_CLIENT_SECRET
      3. `echo <secret_value> | base64 -d`
      4. Confirm secret has propagated, if not repeat steps 1-3 until secret has propagated
   2. Restart application hub
      1. `kubectl rollout restart -n proc deploy/application-hub-hub`
4. Test login through Application Hub

#### Add Users to jupyter-lab Group

In order for users to have access to Jupyter Lab, they must be added to the jupyter-lab group in the Application Hub admin interface. This requires admin privileges within the Application Hub.

1. Log into Application Hub as admin
2. Select Admin tab
3. Manage groups
4. Create jupyter-lab group
5. Add users to group
