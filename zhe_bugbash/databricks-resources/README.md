# Databricks ML Resource Configurations
[(back to main README)](../../README.md)

## Table of contents
* [Intro](#intro)
* [Local development and dev workspace](#local-development-and-dev-workspace)
* [CI & CD](#set-up-ci-and-cd)
* [Develop and test config changes](#develop-and-test-config-changes)

## Intro

### databricks CLI bundles
MLOps-stacks ML resources are configured and deployed through [databricks CLI bundles](https://learn.microsoft.com/azure/databricks/dev-tools/cli/bundle-cli). 
The bundle setting file must be expressed in YAML format and must contain at minimum the top-level bundle mapping.

The databricks CLI bundles top level is defined by file `zhe_bugbash/bundle.yml`.
During databricks CLI bundles deployment, the root config file will be loaded, validated and deployed to workspace provided by the environment together with all the included resources.

ML Resource Configurations in this directory:
 - model workflow (`zhe_bugbash/databricks-resources/model-workflow-resource.yml`)
 - batch inference workflow (`zhe_bugbash/databricks-resources/batch-inference-workflow-resource.yml`)
 - monitoring workflow (`zhe_bugbash/databricks-resources/monitoring-workflow-resource.yml`)
 - feature engineering workflow (`zhe_bugbash/databricks-resources/feature-engineering-workflow-resource.yml`)
 - model definition and experiment definition (`zhe_bugbash/databricks-resources/ml-artifacts-resource.yml`)


### Environment Config & CI/CD integration
The ML resources can be deployed to databricks workspace based on the databricks CLI bundles environment config. 
Different environment configs share the general ML resource configurations with added ability to specify environment specific values(for example, workspace URI, model name, jobs notebook parameters, etc).

This project ships with CI/CD workflows for developing and deploying ML resource configurations based on environment config.

| Environment | Description                                                                                                                                                                                                                           | Databricks Workspace | Model Name                          | Experiment Name                                |
|-------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------|-------------------------------------|------------------------------------------------|
| dev         | `dev` environment is used by ML engineers to deploy ML resources to development workspace with `dev` environment configs. The config is for ML project development purpose.                                                           | dev workspace        | dev-zhe-bugbash-model     | /dev-zhe-bugbash-experiment     |
| staging     | `staging` environment is part of the CD pipeline. Latest main content will be deployed to staging workspace with `staging` environment config.                                                             | staging workspace    | staging-zhe-bugbash-model | /staging-zhe-bugbash-experiment |
| prod        | `prod` environment is part of the CD pipeline. Latest release content will be deployed to prod workspace with `prod` environment config.                                                                      | prod workspace       | prod-zhe-bugbash-model    | /prod-zhe-bugbash-experiment    |
| test        | `test` environment is part of the CI pipeline. For changes targeting at main, upon a PR creation, integration test will be triggered and deploy ML resources to staging workspace with `test` environment. | staging workspace    | test-zhe-bugbash-model    | /test-zhe-bugbash-experiment    |

During ML code development, you can deploy local ML resource configurations together with ML code to databricks workspace and try to run the training, model validation or batch inference pipelines. The deployment will use `dev` environment config by default. 

You can open a PR (pull request) to modify ml code or resource config against main branch. 
The PR will run integration test with `test` environment resource config against staging workspace as well as python unit tests. 

Upon merging a commit to main branch, the main branch content will be deployed to staging workspace with `staging` environment resource configurations.

Upon merging a commit to release branch, the release branch content will be deployed to prod workspace with `prod` environment resource configurations.


![ML resource config diagram](../../docs/images/mlops-resource-config.png)

## Local development and dev workspace

### Set up authentication

Follow the document to learn how databricks CLI authentication works -
[databricks CLI - set up authentication](https://learn.microsoft.com/azure/databricks/dev-tools/cli/bricks-cli#set-up-authentication)

1. Follow [databricks CLI](https://learn.microsoft.com/azure/databricks/dev-tools/cli/bricks-cli) to download and set up databricks CLI locally.
2. Complete the `TODO` sections - add the dev workspace URI to `zhe_bugbash/bundle.yml` under `environments.dev.workspace.host`.
3. [Create a personal access token](https://learn.microsoft.com/azure/databricks/dev-tools/api/latest/authentication#generate-a-personal-access-token)
  in your dev workspace and copy it.
4. Set env parameter `DATABRICKS_TOKEN` with the personal access token in your terminal. For example, run `export DATABRICKS_TOKEN=dapi1234567890ab1cde2f3ab456c7d89efa` if the access token is dapi1234567890ab1cde2f3ab456c7d89efa.
5. Now you can use databricks CLI to validate and deploy ML resource configurations to dev workspace.

Alternatively, you can use other ways described by [databricks CLI](https://learn.microsoft.com/azure/databricks/dev-tools/cli/bricks-cli) to set up authentication. For example, use username/password, or set up profile.

### Validate and provision ML resource configurations
1. After installing databricks CLI and setting up `DATABRICKS_TOKEN`, enter the zhe_bugbash directory when prompted.
2. Run `databricks bundle validate` to validate ML resource configurations. 
3. Run `databricks bundle deploy` to provision ML resource configurations to dev workspace. The ML resource configurations and your ML code will together be copied to dev workspace. Databricks Workflows, Model and Experiment will be provisioned according to the ML resource configs.
4. Run `databricks bundle run <name-of-workflow e.g. model_training_job>` to run a specific workflow defined in your bundle files such as `zhe_bugbash/databricks-resources/model-workflow-resource.yml`.
5. Go to the Databricks dev workspace to check the defined model, experiment, and workflows, and interact with the created workflows.

### Destroy ML resource configurations
After development is done, run `databricks bundle destroy` to destroy(remove) ML resources from the dev workspace. Any model version with `Production` or `Staging` stage will prevent the model from being deleted. Please update the version stage to `None` or `Archived` before destroying the ML resources.

## Set up CI and CD
Please refer to [mlops-setup](../../docs/mlops-setup.md#configure-cicd) for instructions to set up CI and CD.

## Deploy initial ML resources
After completing the prerequisites, create and push a PR branch adding all files to the Git repo:
```
git checkout -b add-ml-resource-config-and-code
git add .
git commit -m "Add ML resource config and ML code"
git push upstream add-ml-resource-config-and-code
```
Open a pull request from the pushed branch. 
CI will run and comment on the PR with a preview of the resources to be deployed.
Once CI passes, merge the PR to deploy an initial set of ML resources.

Then, follow the next section to configure the input and output data tables for the
batch inference job.

### Setting up batch inference job
The batch inference job expects an input Delta table that with a schema that your registered model accepts. To use the batch
inference job, set up such a Delta table in both your staging and prod workspace.
Then, update batch_inference_job base parameters in `zhe_bugbash/databricks-resources/batch-inference-workflow-resource.yml` to pass
the name of the input Delta table and the name of the output Delta table to which to write batch predictions.

As the batch job will be run with the credentials of the service principal that provisioned it, make sure that the service
principal corresponding to a particular environment has permissions to read the input Delta table and modify the output Delta table in that environment's workspace. If the Delta table is in the [Unity Catalog](https://www.databricks.com/product/unity-catalog), these permissions are

* `USAGE` permissions for the catalog and schema of the input and output table.
* `SELECT` permission for the input table.
* `MODIFY` permission for the output table if it pre-dates your job.

### Setting up model validation
The model validation stack focuses on building a plug-and-play stack component for continuous deployment (CD) of models 
in staging and prod.
Its central purpose is to evaluate a registered model and validate its quality before deploying the model to Production/Staging.

Model validation contains three components: 
* [model-workflow-resource.yml](./model-workflow-resource.yml) contain resource config and input parameters for model validation.
* [validation.py](../validation/validation.py) defines custom metrics and validation thresholds that are referenced by above resource config files.
* [notebooks/ModelValidation](../validation/notebooks/ModelValidation.py) contains the validation job implementation. In most cases you don't need to modify this file.

To set up and enable model validation, update [validation.py](../validation/validation.py) to return desired custom metrics and validation thresholds, then 
resolve the TODOs in ModelValidation task of [model-workflow-resource.yml](./model-workflow-resource.yml).

## Develop and test config changes

### databricks CLI bundles schema overview
To get started, open `zhe_bugbash/databricks-resources/batch-inference-workflow-resource.yml`.  The file contains the ML resource definition of
a batch inference job, like:

```$xslt
new_cluster: &new_cluster
  new_cluster:
    num_workers: 3
    spark_version: 12.2.x-cpu-ml-scala2.12
    node_type_id: Standard_D3_v2
    custom_tags:
      clusterSource: mlops-stack/0.1

resources:
  jobs:
    batch_inference_job:
      name: ${bundle.environment}-zhe-bugbash-batch-inference-job
      tasks:
        - task_key: batch_inference_job
          <<: *new_cluster
          notebook_task:
            notebook_path: ../deployment/batch_inference/notebooks/BatchInference.py
            base_parameters:
              env: ${bundle.environment}
              input_table_name: batch_inference_input_table_name
              ...
```

The example above defines a Databricks job with name `${bundle.environment}-zhe-bugbash-batch-inference-job`
that runs the notebook under `zhe_bugbash/deployment/batch_inference/notebooks/BatchInference.py` to regularly apply your ML model
for batch inference. 

At the start of the resource definition, we declared an anchor `new_cluster` that will be referenced and used later. For more information about anchors in yaml schema, please refer to yaml documentation.

We specify a `batch_inference_job` under `resources/jobs` to define a databricks workflow with internal key `batch_inference_job` and job name `{bundle.environment}-zhe-bugbash-batch-inference-job`. 
The workflow contains a single task with task key `batch_inference_job`. The task runs notebook `../deployment/batch_inference/notebooks/BatchInference.py` with provided parameters `env` and `input_table_name` passing to the notebook.
After setting up databricks CLI, you can run command `databricks bundle schema`  to learn more about databricks CLI bundles schema.

The notebook_path is the relative path starting from the resource yaml file.

### Environment config based variables
The `${bundle.environment}` will be replaced by the environment config name during the bundle deployment. For example, during the deployment of a `test` environment config, the job name will be
`test-zhe-bugbash-batch-inference-job`. During the deployment of a `staging` environment config, the job name will be
`staging-zhe-bugbash-batch-inference-job`.


To use different value based on different environment, you can use bundle variables based on environment, for example,
```$xslt
variables:
  batch_inference_input_table: 
    description: The table name to be used for input to the batch inference workflow.
    default: input_table

environments:
  dev:
    variables:
      batch_inference_input_table: dev_table
  test:
    variables:
      batch_inference_input_table: test_table

new_cluster: &new_cluster
  new_cluster:
    num_workers: 3
    spark_version: 12.2.x-cpu-ml-scala2.12
    node_type_id: Standard_D3_v2
    custom_tags:
      clusterSource: mlops-stack/0.1

resources:
  jobs:
    batch_inference_job:
      name: ${bundle.environment}-zhe-bugbash-batch-inference-job
      tasks:
        - task_key: batch_inference_job
          <<: *new_cluster
          notebook_task:
            notebook_path: ../deployment/batch_inference/notebooks/BatchInference.py
            base_parameters:
              env: ${bundle.environment}
              input_table_name: ${var.batch_inference_input_table}
              ...
```
The `batch_inference_job` notebook parameter `input_table_name` is using a bundle variable `batch_inference_input_table` with default value "input_table".
The variable value will be overwritten with "dev_table" for `dev` environment config and "test_table" for `test` environment config:
- during deployment with `dev` environment config, the `input_table_name` parameter will get value "dev_table"
- during deployment with `staging` environment config, the `input_table_name` parameter will get value "input_table"
- during deployment with `prod` environment config, the `input_table_name` parameter will get value "input_table"
- during deployment with `test` environment config, the `input_table_name` parameter will get value "test_table"

### Test config changes
To test out a config change, simply edit one of the fields above, e.g. 
increase cluster size by bumping `num_workers` from 3 to 4. 

Then follow [Local development and dev workspace](#local-development-and-dev-workspace) to deploy the change to dev workspace.
Alternatively you can open a PR. Continuous integration will validate the updated config and display a summary
of the config for review as well as deploy with `test` environment config to staging workspace.

## Deploy config changes

### Dev workspace deployment
Please refer to [Local development and dev workspace](#local-development-and-dev-workspace).

### Test workspace deployment(CI)
After setting up CI&CD, PRs (pull requests) against main branch will trigger CI to run integration test.
The test will deploy model, experiment and workflow resources with `test` environment resource config against staging workspace. Then trigger a run of the model workflow to verify the ML code. 

### Staging and Prod workspace deployment(CD)
When your PR merges to main, continuous deployment automation will deploy `staging` config to staging workspace.

When you about to cut a release, you can create and merge a PR to merge changes from main to release. Continuous deployment automation will deploy `prod` config to prod workspace.

[Back to main project README](../../README.md)
