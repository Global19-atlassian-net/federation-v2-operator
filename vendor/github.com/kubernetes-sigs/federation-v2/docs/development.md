<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Development Guide](#development-guide)
  - [Prerequisites](#prerequisites)
    - [kubernetes](#kubernetes)
    - [docker](#docker)
  - [Adding a new API type](#adding-a-new-api-type)
  - [Running E2E Tests](#running-e2e-tests)
    - [Setup Clusters, Deploy the Cluster Registry and Federation-v2 Control Plane](#setup-clusters-deploy-the-cluster-registry-and-federation-v2-control-plane)
    - [Running Tests](#running-tests)
    - [Running Tests With In-Memory Controllers](#running-tests-with-in-memory-controllers)
    - [Cleanup](#cleanup)
  - [Test Your Changes](#test-your-changes)
    - [Automated Deployment](#automated-deployment)
  - [Test Latest Master Changes (`canary`)](#test-latest-master-changes-canary)
  - [Test Latest Stable Version (`latest`)](#test-latest-stable-version-latest)
  - [Updating Document](#updating-document)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Development Guide

If you would like to contribute to the federation v2 project, this guide will
help you get started.

## Prerequisites

### kubernetes

The federation v2 deployment requires kubernetes version >= 1.11. To see a
detailed list of binaries required, [see the prerequisites section in the
user guide](userguide.md#prerequisites).

### docker

This repo depends on `docker` >= 1.12 to do the docker build work.
Set up your [Docker environment](https://docs.docker.com/install/)

## Adding a new API type

As per the
[docs](http://book.kubebuilder.io/quick_start.html)
for kubebuilder, bootstrapping a new federation v2 API type can be
accomplished as follows:

```bash
# Bootstrap and commit a new type
$ kubebuilder create api --group <your-group> --version v1alpha1 --kind <your-kind>
$ git add .
$ git commit -m 'Bootstrapped a new api resource <your-group>.federation.k8s.io./v1alpha1/<your-kind>'

# Modify and commit the bootstrapped type
$ vi pkg/apis/<your-group>/v1alpha1/<your-kind>_types.go
$ git commit -a -m 'Added fields to <your-kind>'

# Update the generated code and commit
$ make generate
$ git add .
$ git commit -m 'Updated generated code'
```

The generated code will need to be updated whenever the code for a
type is modified. Care should be taken to separate generated from
non-generated code in the commit history.

## Running E2E Tests

The federation-v2 E2E tests must be executed against a deployed
federation of one or more clusters.  Optionally, the federation
controllers can be run in-memory to enable debugging.

Many of the tests validate CRUD operations for each of the federated
types enabled by default:

1. the objects are created in the target clusters.
1. a label update is reflected in the objects stored in the target
   clusters.
1. a placement update for the object is reflected in the target clusters.
1. deleted resources are removed from the target clusters.

The read operation is implicit.

### Setup Clusters, Deploy the Cluster Registry and Federation-v2 Control Plane

In order to run E2E tests, you first need to:

1. Create clusters
   - See the [user guide for a way to deploy clusters](userguide.md#create-clusters)
     for testing federation-v2.
1. Deploy the federation-v2 control plane
   - To deploy the latest version of the federation-v2 control plane, follow
     the [automated deployment instructions in the user guide](userguide.md#automated-deployment).
   - To deploy your own changes, follow the [Test Your Changes](#test-your-changes)
     section of this guide.

Once completed, return here for instructions on running the e2e tests.

### Running Tests

Follow the below instructions to run E2E tests against a test federation.

To run E2E tests for all types:

```bash
cd test/e2e
go test -args -kubeconfig=/path/to/kubeconfig -v=4 -test.v
```

To run E2E tests for a single type:

```bash
cd test/e2e
go test -args -kubeconfig=/path/to/kubeconfig -v=4 -test.v \
    --ginkgo.focus='Federated "secrets"'
```

It may be helpful to use the [delve
debugger](https://github.com/derekparker/delve) to gain insight into
the components involved in the test:

```bash
cd test/e2e
dlv test -- -kubeconfig=/path/to/kubeconfig -v=4 -test.v \
    --ginkgo.focus='Federated "secrets"'
```

### Running Tests With In-Memory Controllers

Running the federation controllers in-memory for a test run allows the
controllers to be targeted by a debugger (e.g. delve) or the golang
race detector.  The prerequisite for this mode is scaling down the
federation controller manager:

1. Reduce the `federation-controller-manager` deployment replicas to 0. This way
   we can launch the necessary federation-v2 controllers ourselves via the test
   binary.

   ```bash
   kubectl scale deployments federation-controller-manager -n federation-system --replicas=0
   ```

   Once you've reduced the replicas to 0, you should see the
   `federation-controller-manager` deployment update to show 0 pods running:

   ```bash
   kubectl -n federation-system get deployment.apps federation-controller-manager
   NAME                            DESIRED   CURRENT   AGE
   federation-controller-manager   0         0         14s
   ```

1. Run tests.

   ```bash
   cd test/e2e
   go test -race -args -kubeconfig=/path/to/kubeconfig -in-memory-controllers=true \
       --v=4 -test.v --ginkgo.focus='Federated "secrets"'
   ```

   Additionally, you can run delve to debug the test:

   ```bash
   cd test/e2e
   dlv test -- -kubeconfig=/path/to/kubeconfig -in-memory-controllers=true \
       -v=4 -test.v --ginkgo.focus='Federated "secrets"'
   ```

### Cleanup

Follow the [cleanup instructions in the user guide](userguide.md#cleanup).

## Test Your Changes

In order to test your changes on your kubernetes cluster, you'll need
to build an image and a deployment config.

**NOTE:** When federation CRDs are changed, you need to run:
```bash
make generate
```
This step ensures that the CRD resources in helm chart are synced.
Ensure binaries from kubebuilder for `etcd` and `kube-apiserver` are in the path (see [prerequisites](#prerequisites)).

### Automated Deployment

If you just want to have this automated, then run the following command
specifying your own image. This assumes you've used the steps [documented
above](#setup-clusters-deploy-the-cluster-registry-and-federation-v2-control-plane) to
set up two `kind` or `minikube` clusters (`cluster1` and `cluster2`):

```bash
./scripts/deploy-federation.sh <containerregistry>/<username>/federation-v2:test cluster2
```

**NOTE:** You can list multiple joining cluster names in the above command.
Also, please make sure the joining cluster name(s) provided matches the joining
cluster context from your kubeconfig. This will already be the case if you used
the steps [documented
above](#setup-clusters-deploy-the-cluster-registry-and-federation-v2-control-plane)
to create your clusters.

## Test Latest Master Changes (`canary`)

In order to test the latest master changes (tagged as `canary`) on your
kubernetes cluster, you'll need to generate a config that specifies the correct
image and generated CRDs. To do that, run the following command:

```bash
make generate
./scripts/deploy-federation.sh <containerregistry>/<username>/federation-v2:canary cluster2
```

## Test Latest Stable Version (`latest`)

In order to test the latest stable released version (tagged as `latest`) on
your kubernetes cluster, follow the
[Automated Deployment](userguide.md#automated-deployment) instructions from the user guide.

## Updating Document

If you are going to add some new sections for the document, make sure to update the table
of contents. This can be done manually or with [doctoc](https://github.com/thlorenz/doctoc).
