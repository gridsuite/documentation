# Development Workflow

## Code Owners

We define ownership of knowledge and responsibilities for code and documentation through specially named `CODEOWNERS` files, which list the usernames of people responsible for a repository. You can find more documentation on [github web site](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)

The definition of owners across all GridSuite repositories is currently in progress.

## Pull Requests

We follow the Git Flow workflow, where all development is done in feature branches and merged to the main branch via pull requests. 

The Pull request  must pass all tests and checks before being allowed to merge.

The pull request must be reviewed and approved by at least one person, and by a code owner before being allowed to merge. As the process of defining code owners is still in progress, the last point applies only for repositories with a CODEOWNERS file.

Owner reviews are not necessarily full functional reviews: for small changes an owner may simply give a formal approval. For larger contributions (new components, API changes, architectural decisions), the review may be more thorough.

The lifecycle of a pull request is as follows:

1. **Author** — when your PR is ready for review, add the **`waiting-for-review`**
   label. This is the signal that it is ready to be picked up by an owner.
2. **Owner assignment** — an owner picks up the PR, **removes** the
   `waiting-for-review` label, and **assigns themselves** to it.
3. **Approvals** — the PR needs two approvals to be mergeable:
  - one from **another developer**, and
  - one from the **owner**.
4. **Merge** — once both approvals are in, the **author (developer) can merge**
   the pull request.

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
