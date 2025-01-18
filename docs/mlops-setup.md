# MLOps Setup Guide
[(back to main README)](../README.md)

## Table of contents
* [Intro](#intro)
* [Create a hosted Git repo](#create-a-hosted-git-repo)
* [Configure CI/CD](#configure-cicd---github-actions)
* [Configure profiles for tests, staging, and prod](#configure-profiles-for-tests-staging-and-prod)
* [Merge PR with initial ML code](#merge-a-pr-with-your-initial-ml-code)
* [Create release branch](#create-release-branch)
* [Deploy ML assets and enable production jobs](#deploy-ml-assets-and-enable-production-jobs)
* [Next steps](#next-steps)

## Intro
This page explains how to productionize the current project, setting up CI/CD and
ML asset deployment, and deploying ML training and inference jobs.

After following this guide, data scientists can follow the [ML Pull Request](ml-pull-request.md) and
[ML Config](../demo/assets/README.md)  guides to make changes to ML code or deployed jobs.

## Create a hosted Git repo
Create a hosted Git repo to store project code, if you haven't already done so. From within the project
directory, initialize Git and add your hosted Git repo as a remote:
```
git init --initial-branch=mlopsmain
```

```
git remote add upstream <hosted-git-repo-url>
```

Commit the current `README.md` file and other docs to the `mlopsmain` branch of the repo, to enable forking the repo:
```
git add README.md docs .gitignore demo/assets/README.md
git commit -m "Adding project README"
git push upstream mlopsmain
```

## Configure CI/CD - GitHub Actions

### Prerequisites
* You must be an account admin to add service principals to the account.
* You must be a Databricks workspace admin in the staging and prod workspaces. Verify that you're an admin by viewing the
  [staging workspace admin console](https://adb-xxxx.xx.azuredatabricks.net#setting/accounts) and
  [prod workspace admin console](https://adb-4079498746205223.3.azuredatabricks.net#setting/accounts). If
  the admin console UI loads instead of the Databricks workspace homepage, you are an admin.

### Set up authentication for CI/CD
#### Set up Service Principal

To authenticate and manage ML assets created by CI/CD, 
[service principals](https://learn.microsoft.com/azure/databricks/administration-guide/users-groups/service-principals)
for the project should be created and added to both staging and prod workspaces. Follow
[Add a service principal to your Azure Databricks account](https://learn.microsoft.com/azure/databricks/administration-guide/users-groups/service-principals#--add-a-service-principal-to-your-azure-databricks-account)
and [Add a service principal to a workspace](https://learn.microsoft.com/azure/databricks/administration-guide/users-groups/service-principals#--add-a-service-principal-to-a-workspace)
for details.

For your convenience, we also have Terraform modules that can be used to [create](https://registry.terraform.io/modules/databricks/mlops-azure-project-with-sp-creation/databricks/latest) or [link](https://registry.terraform.io/modules/databricks/mlops-azure-project-with-sp-linking/databricks/latest) service principals.



#### Configure Service Principal (SP) permissions 
If the created project uses **Unity Catalog**, we expect a catalog to exist with the name of the deployment target by default. 
For example, if the deployment target is dev, we expect a catalog named dev to exist in the workspace. 
If you want to use different catalog names, please update the targets declared in the [demo/databricks.yml](../demo/databricks.yml)
and [demo/assets/ml-artifacts-asset.yml](../demo/assets/ml-artifacts-asset.yml)
 files. 
If changing the staging, prod, or test deployment targets, you'll need to update the workflows located in the .github/workflows directory.

The SP must have proper permission in each respective environment and the catalog for the environments.

For the integration test and the ML training job, the SP must have permissions to read the input Delta table and create experiment and models. 
i.e. for each environment:
- USE_CATALOG
- USE_SCHEMA
- MODIFY
- CREATE_MODEL
- CREATE_TABLE

For the batch inference job, the SP must have permissions to read input Delta table and modify the output Delta table. 
i.e. for each environment
- USAGE permissions for the catalog and schema of the input and output table.
- SELECT permission for the input table.
- MODIFY permission for the output table if it pre-dates your job.


#### Set secrets for CI/CD



After creating the service principals and adding them to the respective staging and prod workspaces, refer to
[Manage access tokens for a service principal](https://learn.microsoft.com/azure/databricks/administration-guide/users-groups/service-principals#--manage-access-tokens-for-a-service-principal)
and [Get Azure AD tokens for service principals](https://learn.microsoft.com/azure/databricks/dev-tools/api/latest/aad/service-prin-aad-token)
to get your service principal credentials (tenant id, application id, and client secret) for both the staging and prod service principals, and [Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
to add the following secrets to GitHub:
- `PROD_AZURE_SP_TENANT_ID`
- `PROD_AZURE_SP_APPLICATION_ID`
- `PROD_AZURE_SP_CLIENT_SECRET`
- `STAGING_AZURE_SP_TENANT_ID`
- `STAGING_AZURE_SP_APPLICATION_ID`
- `STAGING_AZURE_SP_CLIENT_SECRET`
Be sure to update the [Workflow Permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#modifying-the-permissions-for-the-github_token) section under Repo Settings > Actions > General to allow `Read and write permissions`.
  


## Configure profiles for tests, staging, and prod
Address the TODOs in the following files:
* [databricks-dev.yaml](../demo/training/profiles/databricks-dev.yaml): specify recipe configs to use in dev workspace
* [databricks-staging.yaml](../demo/training/profiles/databricks-staging.yaml): specify recipe configs to use in recurring model training and batch inference
  jobs that run in the staging workspace
* [databricks-prod.yaml](../demo/training/profiles/databricks-prod.yaml) specify recipe configs to use in recurring model training and batch inference
  jobs that run in the prod workspace
* [databricks-test.yaml](../demo/training/profiles/databricks-test.yaml): specify recipe configs to use in integration tests(CI)

## Merge a PR with your initial ML code
Create and push a PR branch adding the ML code to the repository.

```
git checkout -b add-ml-code
git add .
git commit -m "Add ML Code"
git push upstream add-ml-code
```

Open a PR from the newly pushed branch. CI will run to ensure that tests pass
on your initial ML code. Fix tests if needed, then get your PR reviewed and merged.
After the pull request merges, pull the changes back into your local `mlopsmain`
branch:

```
git checkout mlopsmain
git pull upstream mlopsmain
```

## Create release branch
Create and push a release branch called `mlopsrelease` off of the `mlopsmain` branch of the repository:
```
git checkout -b mlopsrelease mlopsmain
git push upstream mlopsrelease
git checkout mlopsmain
```

Your production jobs (model training, batch inference) will pull ML code against this branch, while your staging jobs will pull ML code against the `mlopsmain` branch. Note that the `mlopsmain` branch will be the source of truth for ML asset configs and CI/CD workflows.

For future ML code changes, iterate against the `mlopsmain` branch and regularly deploy your ML code from staging to production by merging code changes from the `mlopsmain` branch into the `mlopsrelease` branch.
## Deploy ML assets and enable production jobs
Follow the instructions in [demo/assets/README.md](../demo/assets/README.md) to deploy ML assets
and production jobs.

## Next steps
After you configure CI/CD and deploy training & inference pipelines, notify data scientists working
on the current project. They should now be able to follow the
[ML pull request guide](ml-pull-request.md) and [ML asset config guide](../demo/assets/README.md)  to propose, test, and deploy
ML code and pipeline changes to production.
