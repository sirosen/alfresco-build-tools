# alfresco-build-tools

[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white)](https://github.com/pre-commit/pre-commit)
[![Codacy Badge](https://app.codacy.com/project/badge/Grade/fd8899233e1246c0b48a0684ada35d05)](https://www.codacy.com/gh/Alfresco/alfresco-build-tools/dashboard?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=Alfresco/alfresco-build-tools&amp;utm_campaign=Badge_Grade)

| Build     | Status                                                                                                                                                                      |
|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Travis CI | [![Build Status](https://app.travis-ci.com/Alfresco/alfresco-build-tools.svg?branch=master)](https://app.travis-ci.com/Alfresco/alfresco-build-tools)                       |
| GitHub    | [![CI](https://github.com/Alfresco/alfresco-build-tools/actions/workflows/test.yml/badge.svg)](https://github.com/Alfresco/alfresco-build-tools/actions/workflows/test.yml) |

This repository contains shared/reusable CI configurations for GitHub Actions and Travis mainly to serve the repositories of the Alfresco org but virtually usable by everyone.

Travis should be considered in maintenance mode only since we are actively migrating all of our repos to GitHub Actions. If you are still on Travis or trying to migrate to GitHub Actions, see [Travis section](travis.md).

For security-related topics of GitHub Actions, see the [Security section](security.md).

Here follows the list of GitHub Actions topics available in the current document:

- [alfresco-build-tools](#alfresco-build-tools)
  - [GitHub Actions](#github-actions)
    - [Java setup](#java-setup)
      - [Setup JDK](#setup-jdk)
      - [Setup Maven Credentials](#setup-maven-credentials)
      - [Setup Maven Build Options](#setup-maven-build-options)
  - [GitHub Actions provided by community](#github-actions-provided-by-community)
    - [Auto cancel builds](#auto-cancel-builds)
    - [Docker build and push](#docker-build-and-push)
    - [Docker login](#docker-login)
    - [Retry failing step](#retry-failing-step)
    - [SSH debug](#ssh-debug)
    - [Triggering a workflow in another repository](#triggering-a-workflow-in-another-repository)
  - [GitHub Actions provided by us](#github-actions-provided-by-us)
    - [automate-dependabot](#automate-dependabot)
    - [automate-propagation](#automate-propagation)
    - [configure-git-author](#configure-git-author)
    - [get-branch-name](#get-branch-name)
    - [git-check-existing-tag](#git-check-existing-tag)
    - [get-commit-message](#get-commit-message)
    - [git-commit-changes](#git-commit-changes)
    - [git-latest-tag](#git-latest-tag)
    - [helm-build-chart](#helm-build-chart)
    - [helm-integration-tests](#helm-integration-tests)
    - [helm-package-chart](#helm-package-chart)
    - [helm-parse-next-release](#helm-parse-next-release)
    - [helm-publish-chart](#helm-publish-chart)
    - [helm-release-and-publish](#helm-release-and-publish)
    - [helm-unit-tests](#helm-unit-tests)
    - [helm-update-chart-version](#helm-update-chart-version)
    - [jx-updatebot-pr](#jx-updatebot-pr)
    - [kubectl-keep-nslogs](#kubectl-keep-nslogs)
    - [load-release-descriptor](#load-release-descriptor)
    - [maven-build-and-tag](#maven-build-and-tag)
    - [maven-deploy-file](#maven-deploy-file)
    - [maven-release](#maven-release)
    - [maven-update-pom-version](#maven-update-pom-version)
    - [nexus-create-staging](#nexus-create-staging)
    - [pre-commit](#pre-commit)
    - [rancher](#rancher)
    - [send-slack-notification](#send-slack-notification)
    - [setup-github-release-binary](#setup-github-release-binary)
    - [travis-env-load](#travis-env-load)
    - [update-project-base-tag](#update-project-base-tag)
    - [validate-maven-versions](#validate-maven-versions)
    - [veracode](#veracode)
  - [Reusable workflows provided by us](#reusable-workflows-provided-by-us)
    - [helm-publish-new-package-version.yml](#helm-publish-new-package-versionyml)
  - [Cookbook](#cookbook)
    - [Conditional job/step depending on PR labels](#conditional-jobstep-depending-on-pr-labels)
    - [Serialize pull request builds](#serialize-pull-request-builds)
  - [Known issues](#known-issues)
    - [realpath not available under macosx](#realpath-not-available-under-macosx)
  - [Release](#release)

## GitHub Actions

### Java setup

#### Setup JDK

[actions/setup-java](https://github.com/actions/setup-java) should be used, here is a sample usage:

```yml
      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
          cache: 'maven'
```

#### Setup Maven Credentials

Credentials should be already available via organization secrets, otherwise they would need to be
provided as repository secrets.

Since repositories hold a `settings.xml` file at the root with environment variables `MAVEN_USERNAME` and
`MAVEN_USERNAME` filled for the username and password, only a mapping of variables is needed:

```yml
      - name: Build with Maven
        run: mvn --settings settings.xml [...]
        env:
          MAVEN_USERNAME: ${{ secrets.NEXUS_USERNAME }}
          MAVEN_USERNAME: ${{ secrets.NEXUS_PASSWORD }}
```

Alternatively, the [s4u/maven-settings-action](https://github.com/s4u/maven-settings-action) could be used.

#### Setup Maven Build Options

Maven build options can be shared for a given step on the mvn command line, or extracted as environment variables.

Sample usage:

```yml
      - name: Test with Maven
        run: mvn verify ${{ env.MAVEN_CLI_OPTS }}
        env:
          MAVEN_CLI_OPTS: --show-version -Ddocker.skip -Dlogging.root.level=off -Dspring.main.banner-mode=off
```

When deploying in a second step, these variables can be shared:

```yml
env:
  MAVEN_CLI_OPTS: --show-version -Dlogging.root.level=off -Dspring.main.banner-mode=off

[...]

      - name: Test with Maven
        run: mvn verify ${{ env.MAVEN_CLI_OPTS }}
      - name: Deploy with Maven
        run: mvn deploy ${{ env.MAVEN_CLI_OPTS }} -DskipTests
```

When migrating from Travis, depending on the previous configuration, docker.skip and docker.tag properties might need
to be setup on the command line.

Here is a sample way to extract a branch name that would be used for docker images built with the `build-and-push-docker-images.sh` script, although using the [dedicated action](#docker-build-and-push) can also be
useful.

```yml
      - name: Set stripped branch name as tag
        run: echo "STRIPPED_TAG=$(echo ${{ github.ref_name }} | sed -e 's/[^-_.[:alnum:]]/_/g')" >> $GITHUB_ENV
      - name: Docker Build and Push
        run: sh ./build-and-push-docker-images.sh
        env:
          TAG: ${{ env.STRIPPED_TAG }}
```

## GitHub Actions provided by community

### Auto cancel builds

[This action](https://github.com/styfle/cancel-workflow-action) is a replacement for the Travis settings **Auto cancel branch builds** and **Auto cancel pull request builds**.

### Docker build and push

Consider using this official [Docker action](https://github.com/marketplace/actions/build-and-push-docker-images) for building and pushing containers instead of doing it by hand, for buildx support, caching and more.

### Docker login

Credentials should be already available via organization secrets, otherwise they would need to be
provided as repository secrets.

```yml
      - name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Login to Quay.io
        uses: docker/login-action@v1
        with:
          registry: quay.io
          username: ${{ secrets.QUAY_USERNAME }}
          password: ${{ secrets.QUAY_PASSWORD }}
```

### Retry failing step

[This action](https://github.com/nick-fields/retry) retries an Action step on failure or timeout. Useful for unstable commands or that relies on remote resources that can be flaky sometimes.

### SSH debug

GitHub doesn't provide any native support for SSH debug access to builds like
Travis.

To debug a build is necessary to add when needed a step like the following in
the workflow:

```yml
    - name: Setup tmate session
      uses: mxschmitt/action-tmate@v3
      with:
        # provide access to SSH user that triggered the build
        limit-access-to-actor: true
```

You can also run the step on-demand with a manually triggered build by adding
the `workflow_dispatch` event together with a boolean input:

```yaml
on:
  workflow_dispatch:
    inputs:
      debug_enabled:
        description: Enable SSH debug
        type: boolean
        required: false
        default: false
```

and then invoke the action step conditionally based on this event and input
value:

```yaml
    steps:
    - uses: actions/checkout@v2
    - name: Setup tmate session
      # run only when explicitly requested
      if: ${{ github.event_name == 'workflow_dispatch' && inputs.debug_enabled }}
      uses: mxschmitt/action-tmate@v3
      with:
        # provide access to SSH user that triggered the build
        limit-access-to-actor: true
      # automatically terminate after a given timeout
      timeout-minutes: 30
```

When executing that step, the job will block. If you want to continue with the
following steps, just create a file named `continue` in the current workspace
folder:

```sh
touch continue
```

Please be aware that when the last command of the job finish, also the tmate
session will be terminated automatically, so you may want to add at the end of
the workflow a step like:

```yaml
    # wait for 5 minutes before exiting
    - run: sleep 300
```

### Triggering a workflow in another repository

[actions/github-script](https://github.com/actions/github-script) can be used, here is a sample:

```yml
      - name: Trigger Downstream Builds
        if: steps.is_default_branch.outputs.result == 'true'
        uses: actions/github-script@v5
        with:
          github-token: ${{ secrets.BOT_GITHUB_TOKEN }}
          script: |
            await github.rest.actions.createWorkflowDispatch({
              owner: 'Alfresco',
              repo: 'alfresco-process-connector-services',
              workflow_id: 'build.yml',
              ref: 'develop'
            });
```

Note that this requires using a dedicated token.

Also, the triggered workflow should allow workflow dispatch in its definition (and this configuration should be setup
on the default branch):

```yml
on:
  # allows triggering workflow manually or from other jobs
  workflow_dispatch:
```

## GitHub Actions provided by us

### automate-dependabot

Handles automated approval and merge of dependabot PRs, for minor and patch version updates only:

- automated approval on minor and patch versions
- automated merge on patch versions

This action requires a dedicated secret (named `DEPENDABOT_GITHUB_TOKEN` in the sample) to setup the "auto-merge" behavior: the default `GITHUB_TOKEN` is not used in this case, otherwise a build would not be triggered when the PR is merged, [see reference solution](https://david.gardiner.net.au/2021/07/github-actions-not-running.html).

This secret should be a [dependabot secret](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/managing-encrypted-secrets-for-dependabot), and the token should hold the `repo > repo:status` and `repo > public_repo` scopes for public repositories.
The whole list of "repo" scopes might be needed for the workflow to run ok on private repositories.

```yaml
    - uses: Alfresco/alfresco-build-tools/.github/actions/automate-dependabot@ref
      with:
        token: ${{ secrets.DEPENDABOT_GITHUB_TOKEN }}
```

### automate-propagation

Handles automated approval and merge of propagation PRs used to handle alpha releases on builds.

This action requires a dedicated secret (named `BOT_GITHUB_TOKEN` in the sample) to setup the "auto-merge" behavior: the default `GITHUB_TOKEN` is not used in this case, otherwise a build would not be triggered when the PR is merged, [see reference solution](https://david.gardiner.net.au/2021/07/github-actions-not-running.html).

Another token is also needed to handled approval. It can be the default `GITHUB_TOKEN`, but it cannot be the same one that is used for auto-merge behavior as the user might match the creator of the PR (and auto-approval of a PR is not allowed).

```yaml
    - uses: Alfresco/alfresco-build-tools/.github/actions/automate-propagation@ref
      with:
        auto-merge-token: ${{ secrets.BOT_GITHUB_TOKEN }}
        approval-token: ${{ secrets.GITHUB_TOKEN }}
```

### configure-git-author

Configures the git username and email to associate commits with the provided identity

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/configure-git-author@ref
        with:
          username: ${{ secrets.BOT_GITHUB_USERNAME }}
          email: ${{ secrets.BOT_GITHUB_EMAIL }}
```

### get-branch-name

Loads the name of the branch on which the action was called into `BRANCH_NAME` environment variable

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/get-branch-name@ref
```

### git-check-existing-tag

Checks if a tag with the given name already exists for this remote repository. Returns the output named `exists` with value `'true'` or `'false'`.

```yaml
    - uses: Alfresco/alfresco-build-tools/.github/actions/git-check-existing-tag@ref
      with:
        tag: 1.0.0
```

### get-commit-message

Loads the content of the last commit message that triggered the action into `COMMIT_MESSAGE` environment variable

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/get-commit-message@ref
```

### git-commit-changes

Commits local changes after configuring git user and showing the status of what is going be committed.

```yaml
    - uses: Alfresco/alfresco-build-tools/.github/actions/git-commit-changes@ref
      with:
        username: ${{ secrets.BOT_GITHUB_USERNAME }}
        add-options: -u
        commit-message: "My commit message"
```

### git-latest-tag

Gets the latest tag for the given pattern. The result is returned in the output named `tag`.

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/git-latest-tag@ref
        with:
          pattern: 1.0.0-alpha*
```

### helm-build-chart

Run `helm dep up` and `helm lint` on the specified chart

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/helm-build-chart@ref
        with:
          chart-dir: charts/common
```

### helm-integration-tests

Run `helm upgrade --dryn-run` on the specified chart

```yaml
      - name: Execute dry run
        uses: Alfresco/alfresco-build-tools/.github/actions/helm-integration-tests@ref
        with:
          chart-dir: ${{ env.CHART_DIR }}
          test-rancher-url: ${{ secrets.RANCHER2_URL }}
          test-rancher-access-key: ${{ secrets.RANCHER2_ACCESS_KEY }}
          test-rancher-secret-key: ${{ secrets.RANCHER2_SECRET_KEY }}
          test-cluster-name: ${{ env.TEST_CLUSTER_NAME }}
          test-namespace: ${{ env.TEST_NAMESPACE }}
```

### helm-package-chart

Packages a helm chart into a `.tgz` file and provides the name of the file produced in the output named `package-file`, and its path in the output named `package-file-path`.
The packaged file is also uploaded as an artifact and can be downloaded using `actions/download-artifact`.

```yaml
    - uses: Alfresco/alfresco-build-tools/.github/actions/helm-package-chart@ref
      id: package-helm-chart
      with:
        chart-dir: charts/common
```

### helm-parse-next-release

Parses the next main release version based on the content of Chart.yaml file. The result will be returned using the output named `next-release`.
The suffix `-SNAPSHOT` is removed. For instance, if the version attribute in the Chart.yaml file is `1.0.0-SNAPSHOT`, the result will be `1.0.0`

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/helm-parse-next-release@ref
        id: parse-next-release
        with:
          chart-dir: charts/common
```

### helm-publish-chart

Publishes a new helm chart package (`.tgz`) to a helm chart repository

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/helm-publish-chart@ref
        with:
          helm-charts-repo: Activiti/activiti-cloud-helm-charts
          helm-charts-repo-branch: gh-pages
          chart-package: ${{ steps.package-helm-chart.outputs.package-file-path }}
          token: ${{ secrets.BOT_GITHUB_TOKEN}}
```

### helm-release-and-publish

Releases a new version of a helm chart and publishes it to a helm repository

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/helm-release-and-publish@ref
        with:
          version: 1.0.0
          chart-dir: charts/common
          chart-repository-dir: ${{ env.COMMON_CHART_DIR }}
          helm-repository: Activiti/activiti-cloud-helm-charts
          helm-repository-branch: gh-pages
          helm-repository-token: ${{ secrets.GITHUB_TOKEN }}
          git-username:  ${{ secrets.GITHUB_USERNAME }}
```

### helm-unit-tests

Looks for Helm unit tests written using [helm3-unittest](https://github.com/vbehar/helm3-unittest/blob/master/DOCUMENT.md).

```yaml
     - uses: >-
         Alfresco/alfresco-build-tools/.github/actions/helm-unit-tests@ref
       with:
         chart-dir: charts/myapp
         chart-type: application
```

### helm-update-chart-version

Updates `version` attribute inside `Chart.yaml` file:

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/helm-update-chart-version@ref
        with:
          new-version: 1.0.0
```

### jx-updatebot-pr

Create a Pull Request on each downstream repository using [jx-updatebot](https://github.com/jenkins-x-plugins/jx-updatebot).

Given `.jx/updatebot.yaml` spec in the `alfresco-modeling-service` project:

```yaml
apiVersion: updatebot.jenkins-x.io/v1alpha1
kind: UpdateConfig
spec:
  rules:
    - urls:
        - https://github.com/alfresco/alfresco-process-releases
      reusePullRequest: true
      changes:
        - regex:
            pattern: "version: (.*)"
            files:
              - "docker/quay.io/alfresco/alfresco-modeling-service.yml"
        - regex:
            pattern: "<alfresco-modeling-service.version>(.*)</alfresco-modeling-service.version>"
            files:
              - pom.xml
```

This action will promote alpha version to `alfresco-process-releases` repository via pull request. It will add new commit if there is an existing PR with matching `develop` label.

```yaml
      - name: Promote version
        uses: Alfresco/alfresco-build-tools/.github/actions/jx-updatebot-pr@ref
        with:
          version: ${{ steps.tag.outputs.version }}
          labels: develop
          pull-request-title: "promote(dep): update versions into ${{ github.ref_name }}"
          commit-title: "chore(dep): update ${{ github.repository }} version to ${{ steps.tag.outputs.version }}"
          base-branch-name: ${{ github.ref_name }}
          git-username: ${{ secrets.GIT_USERNAME }}
          git-token: ${{ secrets.GIT_TOKEN }}
          git-author-name: ${{ secrets.GIT_AUTHOR_NAME }}
          git-author-email: ${{ secrets.GIT_AUTHOR_EMAIL }}
```

### kubectl-keep-nslogs

This action allow to collect logs from pods if they are referenced in a deployment, a statefulset or a job.

```yaml
    - name: Upload pods logs
      if: always()
      uses: >-
        Alfresco/alfresco-build-tools/.github/actions/kubectl-keep-nslogs@ref
      with:
        namespace: mynsapp
        log_retention: 7
```

### load-release-descriptor

Used to release Activiti Projects. Load release information from release.yaml file.

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/load-release-descriptor@ref
        id: load-descriptor
        with:
          release-descriptor: release.yaml
```

### maven-build-and-tag

Check out, builds a maven project and docker images, generating a new alpha version for it on push events:

- publish maven artifacts to Nexus
- push docker images to quay.io
- create GitHub tag for the new alpha release

```yaml
    outputs:
      version: ${{ steps.build-and-tag.outputs.version }}
    steps:
      - uses: Alfresco/alfresco-build-tools/.github/actions/maven-build-and-tag@ref
        id: build-and-tag
        with:
          maven-username: ${{ secrets.NEXUS_USERNAME }}
          maven-password: ${{ secrets.NEXUS_PASSWORD }}
          quay-username: ${{ secrets.QUAY_USERNAME }}
          quay-password: ${{ secrets.QUAY_PASSWORD }}
          docker-username: ${{ secrets.DOCKER_USERNAME }}
          docker-password: ${{ secrets.DOCKER_PASSWORD }}
          git-username: ${{ secrets.BOT_GITHUB_USERNAME }}
```

### maven-deploy-file

Upload one or more files to a maven server, without requiring the presence of a
pom.xml. It provides a settings.xml when can find one already configured. When
using a custom settings.xml, you probably want to provide also
`repository-id` that match the credentials id to be used for deploying.

```yaml
      - name: Deploy to Nexus
        uses: Alfresco/alfresco-build-tools/.github/actions/maven-deploy-file@ref
        with:
          group-id: org.alfresco
          artifact-id: custom-alfresco-distribution
          repository-url: https://nexus.alfresco.com/nexus/content/repositories/a-valid-repo/
          version: "1.2.3"
          file: output/build.jar
          classifier: binary
          files: output/build-alt.jar,output/build-alt2.jar
          classifiers: alt,alt2
          types: jar,jar
          maven-username: ${{ secrets.NEXUS_USERNAME }}
          maven-password: ${{ secrets.NEXUS_PASSWORD }}
```

### maven-release

Used to release Activiti projects. Update versions in POM files, create git tags and publish Maven artifacts to staging repository.

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/maven-release@ref
        with:
          repo: Activiti/Activiti
          base-ref: ${{  needs.load-release-info.outputs.activiti-tag }}
          release-version: ${{ needs.load-release-info.outputs.version }}
          staging-repository: ${{ needs.load-release-info.outputs.staging-repository }}
          git-username: ${{ secrets.GITHUB_USERNAME }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          gpg-passphrase: "${{ secrets.GPG_PASSPHRASE }}"
          gpg-secret-keys: "${{ secrets.GPG_SECRET_KEYS }}"
          gpg-owner-trust: "${{ secrets.GPG_OWNERTRUST }}"
          nexus-username: "${{ secrets.NEXUS_USERNAME }}"
          nexus-password: "${{ secrets.NEXUS_PASSWORD }}"
```

### maven-update-pom-version

Updates pom files to the provided version

```yaml
    - uses: Alfresco/alfresco-build-tools/.github/actions/maven-update-pom-version@ref
      with:
        version: 1.0.0-alpha.1
```

### nexus-create-staging

Creates a new staging repository on Nexus, unless there is an existing repository with the same description.
The resulting staging repository will be available in the output named `staging-repository`.

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/nexus-create-staging@ref
        with:
          staging-description: Activiti staging ${{ steps.load-descriptor.outputs.version }}
          nexus-profile-id: "${{ secrets.NEXUS_ACTIVITI7_PROFILE_ID }}"
          nexus-username: "${{ secrets.NEXUS_USERNAME }}"
          nexus-password: "${{ secrets.NEXUS_PASSWORD }}"
```

### pre-commit

You can execute pre-commit step in a dedicated new workflow:

```yml
name: pre-commit

on:
  pull_request:
    branches: [ master ]
  push:

jobs:
  pre-commit:
    runs-on: ubuntu-latest
    steps:
      - uses: Alfresco/alfresco-build-tools/.github/actions/pre-commit@ref
```

or into an existing workflow of your choice just declaring the step:

```yml
      - uses: Alfresco/alfresco-build-tools/.github/actions/pre-commit@ref
```

### rancher

register or detach an EKS cluster to Rancher.
AWS credentials are required only when registering the cluster.

```yaml
      - name: Register Cluster
        uses: Alfresco/alfresco-build-tools/.github/actions/rancher@ref
        with:
          rancher-url: ${{ env.RANCHER2_URL }}
          rancher-access-key: ${{ secrets.RANCHER2_ACCESS_KEY }}
          rancher-secret-key: ${{ secrets.RANCHER2_SECRET_KEY }}
          cluster-name: ${{ env.CLUSTER_NAME }}
          action: "register"
          aws-access-key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: "us-east-2"
```

### send-slack-notification

Sends a slack notification with a pre-defined payload, relying on the [slackapi/slack-github-action](https://github.com/slackapi/slack-github-action) official action.

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/send-slack-notification@ref
        with:
          channel-id: 'channel-id'
          token: ${{ secrets.SLACK_BOT_TOKEN }}
          notification-color: '#A30200'
```

### setup-github-release-binary

[setup-github-release-binary](.github/actions/setup-github-release-binary/action.yml)
Allows the installation of a generic binary from GitHub Releases and add it to the PATH.
See [setup-helm-docs](.github/actions/setup-helm-docs/action.yml) for a usage example.

### travis-env-load

To ease the migration to GitHub Actions of repositories that contains one or
more yaml files containing an `env.global` section of Travis CI. It supports env vars
referencing as value env vars defined early in the file (like Travis does).

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/travis-env-load@ref
        with:
          ignore_regex: ^BRANCH_NAME=.*
          yml_path: .travis/env.yml
```

### update-project-base-tag

Used to update a base tag in the release descriptor. It will add or update the
entry `release.baseTag.$PROJECT` with the value specified in the input `tag`.

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/update-project-base-tag@ref
        with:
          release-descriptor: release.yaml
          project: activiti
          tag: ${{ env.ALPHA_VERSION }}
```

### validate-maven-versions

Validates Maven dependency graph versions to ensure all target includes artifacts versions align

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/validate-maven-versions@ref
        with:
          maven-username: ${{ secrets.NEXUS_USERNAME }}
          maven-password: ${{ secrets.NEXUS_PASSWORD }}
          m2-settings-xml: settings.xml
```

### veracode

Runs Veracode Source Clear Scan

```yaml
      - uses: Alfresco/alfresco-build-tools/.github/actions/veracode@ref
        with:
          srcclr-api-token: ${{ secrets.SRCCLR_API_TOKEN }}
          veracode-fails-build: "false"
```

## Reusable workflows provided by us

### helm-publish-new-package-version.yml

Calculates the new alpha version, creates new git tag and publishes the new package to the helm chart repository

```yaml
  publish:
    uses: Alfresco/alfresco-build-tools/.github/workflows/helm-publish-new-package-version.yml@ref
    needs: build
    with:
      next-version: 7.4.0
      chart-dir: charts/common
      helm-charts-repo: Activiti/activiti-cloud-helm-charts
      helm-charts-repo-branch: gh-pages
    secrets: inherit
```

## Cookbook

This section contains a list of recipes and common patterns organized by desired
outcome.

### Conditional job/step depending on PR labels

A possible approach to have a dynamic behaviour in `pull_request` triggered
workflows, is to check the currently assigned labels. Please be aware that labels
should be already applied before opening/updating a PR in order be effective.

```yml
    if: contains(github.event.pull_request.labels.*.name, 'label-name')
```

### Serialize pull request builds

When a workflow requires to access an external shared resource, it may be
desirable to prevent concurrent builds of the same pull request using
`concurrency` as a top-level keyword:

```yml
name: my-workflow
on:
  pull_request:
    branches:
      - develop
  push:
    branches:
      - develop
concurrency:
  group: ${{ github.head_ref || github.ref_name || github.run_id }}
  cancel-in-progress: false
```

The `github.head_ref` is available when workflow is triggered by pull_request
event, while `github.ref_name` when pushing branches and tags. The
`github.run_id` is just a fallback to avoid failure when both variables are empty.

More docs on [using concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency)

## Known issues

### realpath not available under macosx

When running pre-commit locally you may get failures with the following error:

```sh
realpath: command not found
```

This is because macosx lacks support for that, and it can be fixed with:

```sh
brew install coreutils
```

## Release

Run the release script to release a new version from this repository:

```sh
./release.sh v1.2.3
```
