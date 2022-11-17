# Zupit Reusable Workflows
This repository contains the reusable workflows to check, build, and deploy the web / mobile applications.

Here we list the main workflows to use with the examples of how to use them. Since some workflows are grouped together 
(e.g. the common workflows as you will see), here we skip the details of the *step* workflows already grouped to focus 
only on the most important ones. If you would like to get more details of these tasks, just look at this [doc](docs/GROUPED_STEP_WORKFLOWS.md).

1. [Naming Convention](#naming-convention)
2. [Reusable Workflows](#reusable-workflows)
   1. [Django](#django)
      1. [Common Workflow](#django-common)
   2. [NodeJS](#nodejs)
      1. [Common NodeJS](#nodejs-common)
      2. [Build Docker Image and Push to Registry](#nodejs-build-docker-image-and-push-to-registry)
   3. [Docker](#docker)
      1. [Build Docker Image and Push to Registry](#docker-build-docker-image-and-push-to-registry)
      2. [Deploy Docker Compose](#deploy-docker-compose)
      3. [Delete Docker Images](#delete-docker-images)
   4. [Others](#others)
      1. [Sonar Analyze](#sonar-analyze)

## Naming convention

We define 2 different types of workflows:
- **step**: a *reusable workflow* that *runs a set of specific tasks* that can be grouped together
  (e.g. checking if the project is linted and builds, run the tests, build and push a docker image, ...).
- **workflow**: a *reusable workflow* that *contains a set of our "steps" workflows* to reduce the boilerplate when writing the final workflows.
  One use case is the check if the code is linted and that it builds together with tests, as this is used in almost all our projects.
  The reason why these workflows are not grouped together by default is that for some reasons, tests might not be available.

Our reusable workflows are named to follow this standard:

`<technology-or-application>-<workflow-type>-<action-to-executte>.yml`

Thus, it is easy to understand that the workflows uses a specific technology or application to execute the wanted action.

## Reusable Workflows
In all the examples, we set *secrets: inherit* to pass all secrets to the reusable workflows, but it is also possible to pass a subset of secrets.

In addition, we added for all *step* workflows the input *LABELS* as GitHub does not allow to set the *runs-on* from the caller side, but only inside
the reusable workflows. As we want to define the runners as late as possible, we decided to add this input variable.

In the *workflow* type, you will note that we defined 2 inputs for the labels: NATIVE_LABELS and CONTAINER_LABELS. 
We had to differentiate as GitHub runners might start to raise permissions errors due to Docker being run as root. 
To fix this problem, the workflows using docker images must use different runners from workflows running commands directly on the host.

### Django

#### Django Common
**django-workflow-common.yml** is the reusable workflow to check that the code is correctly linted, that all migrations
are not broken and that all tests pass.

It groups together these reusable workflows:
- *django-step-lint-check.yml*
- *django-step-tests.yml*


It requires these inputs:
- NATIVE_CI_LABELS: the *labels* to select the correct *github-runner* that will execute workflows **WITHOUT** docker. The format is a stringified JSON list of labels.
- CONTAINER_CI_LABELS: the *labels* to select the correct *github-runner* that will execute workflows **WITH** docker. The format is a stringified JSON list of labels.
- WORKING_DIRECTORY: The directory where the runner can execute all the commands.
- PYTHON_IMAGE: The Python Docker image where the runner execute all the commands

In addition, this workflow makes COVERAGE_ARTIFACT_NAME optional:
- COVERAGE_ARTIFACT_NAME: The artifact's name for the *coverage-django.xml* file. By default is **coverage-django.xml**.

This is an example to show how data should be formatted. 
```yaml
jobs:
  django-common:
    uses:
      zupit-it/pipeline-templates/.github/workflows/django-workflow-common.yml@main
    with:
      WORKING_DIRECTORY: backend
      PYTHON_IMAGE: python:3.8.2-slim-buster
      NATIVE_CI_LABELS: "['pinga', 'pipeline', 'native']"
      CONTAINER_CI_LABELS: "['pinga', 'pipeline', 'container']"
      COVERAGE_ARTIFACT_NAME: coverage-django.xml
    secrets: inherit
```

---

### NodeJS
The NodeJS workflows require these commands in order to succeed:
1. **ci:format:check**: Check that the code is formatted correctly.
2. **ci:lint**: Check that the code is linted correctly.
3. **ci:build**: Check that the code builds correctly
4. **ci:e2e**: Check that all tests pass
5. **build:{environment}**: Build the code based on the target **environment** (e.g. *testing*, *staging* and *production*)

#### NodeJS Common
**node-workflow-common.yml** is the reusable workflow to check that the code is correctly formatted and linted, that it
builds correctly and that all tests pass.

It groups together these reusable workflows:
- *node-step-format-lint-build.yml*
- *node-step-test-cypress.yml*

It requires these inputs:
- NATIVE_CI_LABELS: the *labels* to select the correct *github-runner* that will execute workflows **WITHOUT** docker. The format is a stringified JSON list of labels.
- CONTAINER_CI_LABELS: the *labels* to select the correct *github-runner* that will execute workflows **WITH** docker. The format is a stringified JSON list of labels.
- WORKING_DIRECTORY: The directory where the runner can execute all the commands.
- NODE_VERSION: The NodeJS Docker image where the runner execute all the commands
- CYPRESS_IMAGE: The Cypress Docker image where the runner execute all the commands

In addition, it is possible to specify these optional inputs:
- COVERAGE_ARTIFACT_NAME: The artifact's name for the *lcov.info* file. By default, it is **lcov.info**.
- ENABLE_TESTS: Whether it should skip or not the cypress tests workflow. By default, it is **true**.

This is an example to show how data should be formatted. 
```yaml
jobs:
  node-common:
    uses:
      zupit-it/pipeline-templates/.github/workflows/node-workflow-common.yml@main
    with:
      NATIVE_CI_LABELS: "['pinga', 'pipeline', 'native']"
      CONTAINER_CI_LABELS: "['pinga', 'pipeline', 'container']"
      WORKING_DIRECTORY: frontend
      NODE_VERSION: 16.17.0
      CYPRESS_IMAGE: cypress/browsers:node16.17.0-chrome106
    secrets: inherit
```

---

#### NodeJS build docker image and push to registry
**node-step-docker-build-and-push-image.yml** is the workflow that builds the docker image and then push it to the registry.
This is a specific version of the *docker-step-build-and-push-image.yml* as this adds the build of the nodejs project.

*This workflow uses a nodejs docker image, hence remember to use labels to match runners specific for docker.*

It requires these inputs:
- LABELS: the *labels* to select the correct *github-runner* that will execute this workflow. The format is a stringified JSON list of labels.
- NODE_VERSION: The NodeJS version required to build the project.
- WORKING_DIRECTORY: The directory where the runner can execute all the commands.
- RELEASE_ENVIRONMENT: The environment for which the project must be compiled (e.g. *testing*, *staging*, *production*).
- REGISTRY_URL: The registry url where to push the docker image.
- DOCKERFILE_PATH: The path to the dockerfile to build.
- DOCKER_IMAGE_NAME: The name to assign to the built docker image.
- DOCKER_IMAGE_TAG: The tag to assign to the built docker image.
- BUILD_ARGS: Additional data to pass when building the dockerfile.

It then outputs these variables:
- DOCKER_IMAGE_NAME: The final docker image name with the registry path included.

This is an example to show how data should be formatted. 
```yaml
jobs:
  build-and-push-image:
    uses: 
      zupit-it/pipeline-templates/.github/workflows/node-step-docker-build-and-push-image.yml@main
    with:
      LABELS: "['pinga', 'pipeline', 'container']"
      NODE_VERSION: '16.17.0'
      RELEASE_ENVIRONMENT: testing
      WORKING_DIRECTORY: frontend
      REGISTRY_URL: ghcr.io
      DOCKERFILE_PATH: frontend/docker/Dockerfile
      DOCKER_IMAGE_NAME: ionic
      DOCKER_IMAGE_TAG: latest
      BUILD_ARGS: |
        DIST_PATH=dist/apps/enci
    secrets: inherit
```

---

### Docker

#### Docker build docker image and push to registry
**docker-step-build-and-push-image.yml** is the workflow that builds the docker image and then push it to the registry.

It requires these inputs:
- LABELS: the *labels* to select the correct *github-runner* that will execute this workflow. The format is a stringified JSON list of labels.
- WORKING_DIRECTORY: The directory where the runner can execute all the commands.
- RELEASE_ENVIRONMENT: The environment for which the project must be compiled (e.g. *testing*, *staging*, *production*).
- REGISTRY_URL: The registry url where to push the docker image.
- DOCKERFILE_PATH: The path to the dockerfile to build.
- DOCKER_IMAGE_NAME: The name to assign to the built docker image.
- DOCKER_IMAGE_TAG: The tag to assign to the built docker image.
- BUILD_ARGS: Additional data to pass when building the dockerfile.

It then outputs these variables:
- DOCKER_IMAGE_NAME: The final docker image name with the registry path included.

This is an example to show how data should be formatted. 
```yaml
jobs:
  build-and-push-image:
    uses: 
      zupit-it/pipeline-templates/.github/workflows/docker-step-build-and-push-image.yml@main
    with:
      LABELS: "['pinga', 'pipeline', 'native']"
      RELEASE_ENVIRONMENT: testing
      WORKING_DIRECTORY: backend
      REGISTRY_URL: ghcr.io
      DOCKERFILE_PATH: backend/docker/Dockerfile
      DOCKER_IMAGE_NAME: django
      DOCKER_IMAGE_TAG: latest
    secrets: inherit
```

---

#### Deploy Docker Compose
**docker-step-deploy.yml** is the workflow that starts a docker compose file on the targeted host.

It requires these inputs:
- LABELS: the *labels* to select the correct *github-runner* that will execute this workflow. The format is a stringified JSON list of labels.
- ENVIRONMENT: The target environment that will show GitHub on the GitHub action page.
- DEPLOY_URL: The target environment url that will show GitHub on the GitHub action page. 
- REGISTRY_URL: The registry url where to pull the images.
- PROJECT_NAME: The name that will be associated to the Docker Compose stack.
- DOCKER_COMPOSE_PATH: The path to the docker compose to start.
- IMAGES: A stringified json object containing as key the environment variables images used in the 
  docker compose file and as value the name of the images that will be downloaded from the registry.
  You can retrieve dynamically the image name from the *docker build and push step* by adding the step's name to the **needs** array of the workflow 
  and using `${{ needs.{STEP_NAME}.outputs.DOCKER_IMAGE_NAME }}` where STEP_NAME is the step's name. 

This is an example to show how data should be formatted. 
```yaml
jobs:
  deploy:
    uses: 
      zupit-it/pipeline-templates/.github/workflows/docker-step-deploy.yml@main
    with:
      LABELS: "[ 'pinga', 'deploy', 'native', 'zupit-applications' ]"
      ENVIRONMENT: testing
      DEPLOY_URL: https://workflows-example.testing.zupit.software
      REGISTRY_URL: ghcr.io
      PROJECT_NAME: workflows-example
      DOCKER_COMPOSE_PATH: docker/testing/docker-compose.yml
      IMAGES: "{
          'BACKEND_IMAGE_TAG': '${{ needs.backend-step.outputs.DOCKER_IMAGE_NAME }}',
          'FRONTEND_IMAGE_TAG': '${{ needs.frontend-step.outputs.DOCKER_IMAGE_NAME }}'
        }"
    secrets: inherit
```

---

#### Delete Docker Images
**docker-step-delete-images.yml** is the workflow that manage the retention policy on the GitHub Registry. 
It deletes all untagged images and it allows to have a maximum of N tagged images for staging and other N tagged images for production.
This workflow should be scheduled using cron to achieve the retention policy.

The images' tags must follow this standard:
- `latest`: for testing environment. This won't be deleted.
- `v[0-9]+.[0-9]+.[0-9]+-rc`: for staging environment.
- `v[0-9]+.[0-9]+.[0-9]+`: for production environment.

It requires these inputs:
- LABELS: the *labels* to select the correct *github-runner* that will execute this workflow. The format is a stringified JSON list of labels.
- IMAGE_NAME: The image name to apply the retention policy.
- KEEP_AT_LEAST: The number of tagged version to maintain for both staging and production environments.

It also requires these secrets:
- RETENTION_POLICY_TOKEN: A PAT with permissions to **read:packages** and **delete:packages** 

In addition, it is possible to specify these optional inputs:
- DRY_RUN: Only for tagged images, it shows which ones will be deleted. 

This is an example to show how data should be formatted. 
```yaml
jobs:
  clean-ionic-images:
    uses: 
      ZupitSRL/pipeline-templates/.github/workflows/docker-step-delete-images.yml@main
    with:
      LABELS: "['pinga', 'pipeline', 'native']"
      IMAGE_NAME: 'ionic'
    secrets: inherit
```

---

### Others

#### Sonar Analyze
**sonar-step-analyze.yml** is the workflow that analyze the coverage and sends the results to Sonarqube.

It requires these inputs:
- LABELS: the *labels* to select the correct *github-runner* that will execute this workflow. The format is a stringified JSON list of labels.
- WORKING_DIRECTORY: The directory where the runner can execute all the commands.

It also requires these secrets:
- SONAR_TOKEN: The Sonarqube token.

In addition, it is possible to specify these optional inputs:
- SONAR_IMAGE: The Sonarqube docker image where the runner execute all commands. By default, it is **sonarsource/sonar-scanner-cli**.
- SONAR_HOST_URL: The Sonarqube host to where submit analyzed data. By default, it is **https://sonarqube.zupit.software**
- DOWNLOAD_ARTIFACT: Whether it should download an artifact or not to analyze. By default, it is **true**.
- ARTIFACT_FILENAME: The name of the artifact. By default, it is an empty string.

This is an example to show how data should be formatted. 
```yaml
jobs:
  angular-sonar-analyze:
    uses: 
      zupit-it/pipeline-templates/.github/workflows/sonar-step-analyze.yml@main
    with:
      WORKING_DIRECTORY: frontend
      ARTIFACT_FILENAME: lcov.info
      LABELS: "['pinga', 'pipeline', 'container']"
    secrets: inherit
```