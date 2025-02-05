# Contributing Code or Documentation

To work on this project, you need a Kubernetes cluster accessible via `kubectl`. It can be a local cluster based on
[Minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/),
[Kind](https://kind.sigs.k8s.io/) or even a remote cluster hosted
on AWS / GCP / Azure / Oracle Cloud / etc.

## Setting up the environment

### Minikube
When using Minikube, run command below which will configure the docker environment to use the minikube docker runtime
than the system one. This is needed in order to successfully setup the environment:
```shell script
eval $(minikube -p minikube docker-env)
```

Now build the example services docker images:
```shell script
./gradlew clean dockerBuild --refresh-dependencies
```

### Kind
When using [Kind](https://kind.sigs.k8s.io/), use existing or create new local cluster by executing:

```shell script
kind create cluster --wait 5m
```

There is a script that will take care to create the example services docker images used in tests and load them into the cluster:

```shell script
./setup-kubernetes.sh
```

Note: In case you want to use other than default Kind cluster name `kind`, run the setup script with cluster name as firdt parameter:

```shell script
./setup-kubernetes.sh <CLUSTER_NAME>
```

## Running all the tests

The test suite is composed of:

* Few unit tests.
* Integration tests that spin a local Micronaut instance, grab a bean from the context, invoke methods and make
  assertions. Those use either `@MicronautTest` or `ApplicationContext.run()`.
* Functional tests that exercise the sample applications deployed in the Kubernetes cluster. They are all the tests located
  inside `examples` directory.
  
To run all the tests, simply execute:

`./gradlew test`

## Sample applications

The test infrastructure is composed of 3 sample applications that will be deployed to Kubernetes:

* `example-service`: contains some endpoints to check distributed configuration. There are 2 replicas of it (for 
  load-balancing testing purposes). It is accessible locally at the following ports:
  * `9999` and `9998` to access the application for each of the replicas.
  * `5004` to attach a remote debugger.

* `example-client`: it acts as a service discovery client for `example-service`. It is accessible locally at the 
  following ports:
  * `8888` to access the application.
  * `5005` to attach a remote debugger.

* `example-kubernetes-client`: contains endpoints to check the official Kubernetes client integration
  * `8082` to access the application.
  * `5005` to attach a remote debugger.

The deployment manifests are located in the `test-utils` module's resource directory.

## Writing integration tests
When writing tests for new module start by adding dependency to `test-utils` module:

```
    testImplementation project(":test-utils")
```

Then setup a test specification:

```groovy
@MicronautTest
@Requires({ TestUtils.kubernetesApiAvailable() })
@Property(name = "kubernetes.client.namespace", value = "kubernetes-micronaut")
@Property(name = "spec.reuseNamespace", value = "false")
class ApiClientFactorySpec extends KubernetesSpecification {

}
```

The `KubernetesSpecification` class contains the instrumentations for the programatic setup of the Kubernetes namespace that is specified by the `kubernetes.client.namespace` property. The `KubernetesSpecification` will take care of creating the namespace. If you want to always re-create the namespace set property `spec.reuseNamespace` to `false` otherwise if the namespace exists the setup of resources will be skipped.

If you want to setup other kind resources just override the `setupFixture` method:

```groovy
    @Override
    def setupFixture(String namespace) {
        createNamespaceSafe(namespace)
        operations.createRole("kubernetes-client", namespace)
        operations.createRoleBinding("kubernetes-client", namespace, "kubernetes-client")
        operations.createDeploymentFromFile(loadFileFromClasspath("k8s/kubernetes-client-example-deployment.yml"), "kubernetes-client-example", namespace)
        operations.createService("kubernetes-client-example", namespace,
                new ServiceSpecBuilder()
                        .withType("LoadBalancer")
                        .withPorts(
                                new ServicePortBuilder()
                                        .withPort(8085)
                                        .withTargetPort(new IntOrString(8085))
                                        .build()
                        )
                        .withSelector(["app": "kubernetes-client-example"])
                        .build())
    }
```


## Checkstyle

Be aware that this project uses Checkstyle, and your PR might fail in Travis if you do not pay attention to any potential
Checkstyle warnings. Make sure you run Checkstyle for your branch before submitting any PR:

```shell script
./gradlew checkstyleMain
```
