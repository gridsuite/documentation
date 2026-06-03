# Development Workflow

## Pull Requests

We follow the Git Flow workflow, where all development is done in feature branches and merged to the main branch via pull requests. Pull requests are reviewed by at least one other developer before being merged and must pass all tests and checks before being allowed to merge.

## Build and Test

All repositories are configured to build, run unit tests, and perform checks on pull requests using GitHub Actions.

The checks include code quality analysis via SonarCloud (https://sonarcloud.io/organizations/gridsuite/projects), which is configured to run on pull requests and report code quality issues and coverage metrics.

The checks also include Checkstyle for Java code and ESLint for frontend code. The Checkstyle configuration is inherited from PowSyBl.

The build process creates Docker images which are pushed to Docker Hub with the `latest` tag when a commit is made on main (normally after merging a pull request).

Across repositories, CI workflows in `.github` follow a common pattern and call a shared PowSyBl GitHub Action for builds.

## Deployment

### Demo Deployment

A demo deployment is available at https://demo.gridsuite.org/.

The deployment is configured in the "deployment" repository (https://github.com/gridsuite/deployment), which contains the configuration files for deploying the application on a Kubernetes cluster on Azure.

The deployment process is triggered whenever a branch is merged in a component repository. The triggering is done using an action that broadcasts an event (https://github.com/gridsuite/broadcast-event) to the deployment repository when the build is done on the main branch of a component.

The deployment is done using Kustomize.

The demo deployment does not deploy the GridAdmin module, so there is no authorization mechanism on the demo (every account can do everything).

### Docker Compose

The "deployment" repository provides a Docker Compose configuration that can be used locally by developers.

## Release Process

Releases are done via GitHub Actions using the aggregator repository (https://github.com/gridsuite/aggregator), which creates a version tag on all the main branches and then triggers a release process that builds and pushes the Docker images to Docker Hub with the version tag.

A manual release process is done to push libraries to npm and Maven Central. The libraries have a lifecycle independent of the Docker components.

## Utilities

To ease the work of developers working across multiple repositories, an "aggregator" permits managing all GridSuite repositories as submodules. It provides a single entry point to clone, update, and build all repositories, and to review and merge PRs across them. See https://github.com/gridsuite/aggregator.
