# Istio Pilot testing infrastructure

All contributions to Istio Pilot should supply tests that cover new features and/or bug fixes.
We strive to improve and maintain quality of the code with up-to-date samples, good coverage of the code, and static analyzers to detect problems early.

## Getting started

Pilot tests require access to a Kubernetes cluster (version 1.5.2 or higher). Please
configure your `kubectl` to point to a development cluster (e.g. minikube)
before building or invoking the tests and add a symbolic link to your
repository pointing to Kubernetes cluster credentials:

    ln -s ~/.kube/config platform/kube/

To run the complete set of tests, use the following commands:

    bazel test //...
    bin/e2e.sh [-hub docker.io/<username>]
    
_Note1_: If you are running Bazel in a VM (e.g. in [Vagrant environment](build-vagrant.md)), copy
the kube config file on the host to platform/kube instead of symlinking it,
and change the paths to minikube certs.

    cp ~/.kube/config platform/kube/
    sed -i 's!/Users/<username>!/home/ubuntu!' platform/kube/config

Also, copy the same file to `/home/ubuntu/.kube/config` in the VM, and make
sure that the file is readable to user `ubuntu`.

_Note2_: If you are using GKE, please make sure you are using static client
certificates before fetching cluster credentials:

    gcloud config set container/use_client_certificate True

_Note3_: The optional `-h` flag should point to a Docker registry that you have access to push images.

## Code linters

We require that Istio Pilot code contributions pass all linters defined by [the check script](../bin/check.sh):

    bin/check.sh
    
_Note_: You need to set up Go-compatible build environment first, since the linters do not use Bazel. 

## Unit tests

We follow Golang unit testing practice of creating `source_test.go` next to `source.go` files that provide sufficient coverage of the functionality. These tests can also be run using `bazel` with additional strict dependency declarations.

For tests that require Kubernetes access, we rely on the client libraries and `.kube/config` file that needs to be linked into your repository directory as `platform/kube/config`. Kubernetes tests use this file to authenticate and access the cluster.
Each test creates a temporary namespace and deletes it on completion.

For tests that require systems integration, such as invoking the proxy with a special configuration, we capture the desired output as golden artifacts and save the artifacts in the repository. Validation tests compare generated output against the desired output. For example, [Envoy configuration test data](../proxy/envoy/testdata) contains auto-generated proxy configuration. If you make changes to the config generation, you also need to create or update the golden artifact in the same pull request. The test library can automatically refresh all golden artifacts if you pass a special environment variable:

    env REFRESH_GOLDEN=true go test -v istio.io/pilot/...

## Integration tests

Istio Pilot runs end-to-end tests as part of the presubmit check. The test driver is a [Golang program](../test/integration) that creates a temporary namespace, deploys Istio components, send requests from apps in the cluster, and checks that traffic obeys the desired routing policies. The end-to-end test is entirely hermetic: test applications and Istio Pilot docker images are generated on each run. This means you need to have a docker registry to host your images, which then needs to be passed with `-hub` flag. The test driver is invoked using [the e2e script](../bin/e2e.sh).

## Docker images

The following Bazel command generates Docker images for the proxy agent and proxy container:

    bazel run //docker:runtime

Istio Pilot also produces debug images in addition to the default bare images. These images have suffix `_debug` and include additional tools such as `curl` in the base image as well as debug-enabled Envoy builds. You might need to grant security privileges to the container spec for root access:

    securityContext:
      privileged: true

The proxy injection process redirects *all* inbound and outbound traffic through
the proxy via iptables. This can sometimes be undesirable while debugging, e.g.
trying to install additional test tools via apt-get. Use
`proxy-redirection-clear` to temporarily disable the iptable redirection rules
and `proxy-redirection-restore` to restore them.

## Test logging

Istio Pilot uses [glog](https://godoc.org/github.com/golang/glog) library for all its logging. We encourage extensive logging at the appropriate log levels. As a hint to the log level selection, level 10 is the most verbose (Kubernetes will show all its HTTP requests), level 2 is used by default in the integration tests, level 4 turns on extensive logging in the proxy.

