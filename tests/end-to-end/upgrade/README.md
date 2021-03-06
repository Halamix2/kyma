# End-to-end Upgrade Tests

## Overview

This project contains end-to-end upgrade tests run for the Kyma [upgrade plan](https://github.com/kyma-project/test-infra/blob/master/prow/scripts/cluster-integration/kyma-gke-upgrade.sh) on CI. The tests are written in Go. The framework allows you to define two actions:

- Preparing the data 
- Running tests against the prepared data

## Prerequisites

To set up the project, use these tools:

- Version 1.11 or higher of [Go](https://golang.org/dl/)
- Version 0.5 or higher of [Dep](https://github.com/golang/dep)
- The latest version of [Docker](https://www.docker.com/)

## Usage

Run end-to-end upgrade tests for the Kyma [upgrade plan](https://github.com/kyma-project/test-infra/blob/master/prow/scripts/cluster-integration/kyma-gke-upgrade.sh) on Prow. The continuous integration flow looks as follows:

1. Install Kyma from the [latest](https://github.com/kyma-project/kyma/releases/latest) release.
2. Install the upgrade-test [helm chart](chart/upgrade). 
3. Execute all TestDefinitions with the label `kyma-project.io/test.before-upgrade`.
4. Upgrade the Kyma cluster.
5. Execute all TestDefinitions with the label `kyma-project.io/test.after-upgrade`.

### Use environment variables

Use the following environment variables to configure the application:

| Name | Required | Default | Description |
|-----|:---------:|--------|------------|
| **APP_LOGGER_LEVEL** | NO | `info` | A parameter that sets the logging level in an application. The possible values are `debug`, `info`, `warn`, `warning`, `error`, `fatal`, and `panic`. |
| **APP_KUBECONFIG_PATH** | NO | None | A path to the `kubeconfig` file needed to run an application outside of the cluster. |
| **APP_MAX_CONCURRENCY_LEVEL** | NO | `1` | A maximum concurrency level used for running tests. |
| **APP_TESTING_ADDONS_URL** | YES | None | An external link to testing addons. |
| **APP_WORKING_NAMESPACE** | No | `e2e-upgrade-test` | A Namespace in which the test is run. |
| **APP_TESTS_INFO_CONFIG_MAP_NAME** | No | `upgrade-tests-info` | A name of a ConfigMap in which the upgrade test framework stores information about passed tests. |

### Use flags

Use the following flags to configure the application:

| Name | Required | Description |
|-----|:---------:|------------|
| **action** | YES | Defines what kind of action to execute. The possible values are `prepareData` and `executeTests`. |
| **verbose** | NO | Prints logs for all tests. |
| **testName** | NO | Limit test execution to the given test. If not specified, all tests will be executed. |

See the example:

```go
go run main.go --action prepareData --verbose
```

## Development

>**NOTE:** The following approach for adding new tests to the upgrade scenario is not needed anymore. To prepare data during the preparation phase, create a new TestDefinition that executes your preparation code and add the label `kyma-project.io/before-upgrade=true`. To execute evaluation code during the evaluation phase, create a new TestDefinition with the label `kyma-project.io/after-upgrade=true` 


This section presents how to add and run a new test. It also describes how to verify the code and ensure that your test is correct.

### Add a new test

Add a new test under the `pkg/tests/{domain-name}` directory and implement the following interface:

```go
    UpgradeTest interface {
        CreateResources(stop <-chan struct{}, log logrus.FieldLogger, namespace string) error
        TestResources(stop <-chan struct{}, log logrus.FieldLogger, namespace string) error
    }
```

In each method, the framework injects the following parameters:

- **stop** - a channel called when the application shutdown is requested. Use it to gracefully shutdown your test.
- **log** - a logger used when you need additional logging in your test. Logged data is printed only if a given method fails.
- **namespace** - a space created for you by the framework.

This interface allows you to easily register the test in the [`main.go`](./main.go) file by adding a new entry in the test map:

```go
  // Register tests. Convention:
  // {test-name} : {test-instance}
  //
  // Using map is intentional - we ensure that test name is not duplicated.
  // Test name is sanitized and used for creating dedicated Namespace for a given test
  // so that it doesn't overlap with others.
  tests := map[string]runner.UpgradeTest{
    "YourTestName": yourpkg.NewTest(),
  }
```

See the example test [here](./pkg/tests/hello-world/test.go).

>**NOTE:**  If your test operates on Kubernetes resources, ensure that RBAC rules are updated in the [upgrade](./chart/upgrade) Helm chart.

### Run end-to-end tests locally

Run the application without building a binary file. To do so:

1. Prepare the upgrade data:

   ```bash
   kubectl create configmap tests-info -n kyma-system
   env APP_WORKING_NAMESPACE=kyma-system APP_TESTS_INFO_CONFIG_MAP_NAME=tests-info APP_KUBECONFIG_PATH=/Users/$USER/.kube/config APP_LOGGER_LEVEL=debug APP_TESTING_ADDONS_URL="https://github.com/kyma-project/addons/releases/download/0.8.0/index-testing.yaml" go run main.go --action prepareData --testName $YOURTESTNAME
   ```

2. Run tests:

   ```bash
   env APP_WORKING_NAMESPACE=kyma-system APP_TESTS_INFO_CONFIG_MAP_NAME=tests-info APP_KUBECONFIG_PATH=/Users/$USER/.kube/config APP_LOGGER_LEVEL=debug APP_TESTING_ADDONS_URL="https://github.com/kyma-project/addons/releases/download/0.8.0/index-testing.yaml" go run main.go --action executeTests --testName $YOURTESTNAME
   ```

For the description of the available environment variables, see [this](#use-environment-variables) section.

### Run tests using a Helm chart

Run the application using Helm:

1. Prepare the upgrade data:

    ```bash
    helm install --name e2e-test-upgrade --namespace {namespace} ./chart/upgrade/ --wait --tls
    ```

2. Run preparation phase:

    ```bash
    kyma test run -l kyma-project.io/before-upgrade=true
    ```

3. Run evaluation phase:

    ```bash
    kyma test run -l kyma-project.io/after-upgrade=true
    ```

### Run tests using Telepresence

[Telepresence](https://www.telepresence.io/) allows you to run tests locally while connecting a service to a remote Kubernetes cluster. It is helpful when the test needs access to other services in a cluster.

1. [Install Telepresence](https://www.telepresence.io/reference/install).
2. Run tests:

   ```bash
   env APP_KUBECONFIG_PATH=/Users/$USER/.kube/config APP_LOGGER_LEVEL=debug APP_TESTING_ADDONS_URL="https://github.com/kyma-project/addons/releases/download/0.8.0/index-testing.yaml" telepresence --run go run main.go  --action executeTests --testName $YOURTESTNAME --verbose
   ```

### Verify the code

Use the `make verify` command to test your changes before each commit. To build an image, use the `make build-image` command with **DOCKER_PUSH_REPOSITORY** and **DOCKER_PUSH_DIRECTORY** variables, for example:

```bash
DOCKER_PUSH_REPOSITORY=eu.gcr.io DOCKER_PUSH_DIRECTORY=/kyma-project/develop make build-image
```

### Project structure

The repository has the following structure:

```text
.
├── chart                   # The Helm chart for deploying the upgrade test application
├── internal                # The internal source code of the upgrade test framework
├── pkg                     # The directory which contains all secondary Go packages
│   └── tests               # The package where upgrade tests are defined. Put your test here.
├── vendor                  # Dep-managed dependencies
├── main.go                 # The entrypoint for upgrade test runner
├── Gopkg.toml              # A dep manifest
└── Gopkg.lock              # A dep lock which is generated automatically. Do not edit it.
```
