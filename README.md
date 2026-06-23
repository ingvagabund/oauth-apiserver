# oauth-apiserver

## Overview

The OpenShift oauth-apiserver is an [aggregated API server](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/) that is responsible
for serving some of OpenShift's authentication and authorization related APIs.

It also serves as a [Kubernetes Webhook Token Authenticator](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication).

### Aggregated APIs served

The oauth-apiserver serves both the `oauth.openshift.io` and `user.openshift.io` API groups.

More details on the APIs in these groups can be found in the OpenShift documentation:
- [oauth.openshift.io API Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/oauth_apis/oauth-apis)
- [user.openshift.io API Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/user_and_group_apis/user-and-group-apis)

Most of the user-facing API definitions for these groups will be found in the [openshift/api repository](https://github.com/openshift/api)

## Contributing

For guidance on contributing, see [CONTRIBUTING.md](CONTRIBUTING.md)

## Building

### Binary

To build the oauth-apiserver binary, run the following command:
```sh
make build
```

### Image

To build an image for the oauth-apiserver:

1. Log in to the [`app.ci` cluster](https://console-openshift-console.apps.ci.l2s4.p1.openshiftapps.com/) using Red Hat SSO.
2. Click on your username and then click on "Copy login command".
3. Click "Display Token".
4. Copy your token.
5. Run `podman login registry.ci.openshift.org`.
    - For your username, use your Kerberos ID (the same username shown in the OpenShift console for the `app.ci` cluster).
    - For your password, use the token you copied.
6. Run `podman build -f images/Dockerfile.rhel7 -t ${IMAGE_TAG} .`
    - If you are using MacOS, you'll need to set the platform with the `--platform linux/amd64` flag to run the image on an OpenShift cluster.

## Testing

### Unit tests

To run the unit tests for the entire project, run:

```sh
make test
```

### End-to-end tests

This repository is compatible with the [OpenShift Tests Extension (OTE)](https://github.com/openshift-eng/openshift-tests-extension) framework.

#### Building the test binary

```bash
make build
```

#### Running test suites and tests

```bash
# Run a specific test suite or test
./oauth-apiserver-tests-ext run-suite "openshift/oauth-apiserver/all"
./oauth-apiserver-tests-ext run-test "test-name"

# Run with JUnit output
./oauth-apiserver-tests-ext run-suite openshift/oauth-apiserver/all --junit-path /tmp/junit.xml
```

#### Listing available tests and suites

```bash
# List all test suites
./oauth-apiserver-tests-ext list suites

# List tests in a suite
./oauth-apiserver-tests-ext list tests --suite=openshift/oauth-apiserver/all
```

For more information about the OTE framework, see the [openshift-tests-extension documentation](https://github.com/openshift-eng/openshift-tests-extension).

Beyond the OTE tests, there are some additional e2e tests that can be run with `make test-e2e`.
