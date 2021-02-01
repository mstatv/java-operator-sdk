# ![java-operator-sdk](docs/assets/images/logo.png) 
![Java CI with Maven](https://github.com/java-operator-sdk/java-operator-sdk/workflows/Java%20CI%20with%20Maven/badge.svg)

Build Kubernetes Operators in Java without hassle. Inspired by [operator-sdk](https://github.com/operator-framework/operator-sdk).

| S.No. | Contents |
| ----- | -------- |
| [1.](#Features) | [Features](#Features) |
| [2.](#Why-build-your-own-Operator) | [Why build your own Operator?](#Why-build-your-own-Operator) |
| [3.](#Roadmap) | [Roadmap](#Roadmap) |
| [4.](#Join-us-on-Discord) | [Join us on Discord!](#Join-us-on-Discord) |
| [5.](#User-Guide) | [User Guide](#User-Guide) |
| [6.](#Usage) | [Usage](#Usage) |
| [7.](#Spring-Boot) | [Spring Boot](#Spring-Boot) |

#### Features
* Framework for handling Kubernetes API events
* Automatic registration of Custom Resource watches
* Retry action on failure
* Smart event scheduling (only handle the latest event for the same resource)

Check out this [blog post](https://blog.container-solutions.com/a-deep-dive-into-the-java-operator-sdk) 
about the non-trivial yet common problems needed to be solved for every operator. 

#### Why build your own Operator?
* Infrastructure automation using the power and flexibility of Java. See [blog post](https://blog.container-solutions.com/cloud-native-java-infrastructure-automation-with-kubernetes-operators).
* Provisioning of complex applications - avoiding Helm chart hell
* Integration with Cloud services - e.g. Secret stores
* Safer deployment of applications - only expose cluster to users by Custom Resources

#### Roadmap
* Testing of the framework and all samples while running on a real cluster.
* Generate a project skeleton
* Generate Java classes from CRD defintion (and/or the other way around)
* Integrate with Quarkus (including native image build)
* Integrate with OLM (Operator Lifecycle Manager)

#### Join us on Discord!

[Discord Invite Link](https://discord.gg/DacEhAy)

#### User Guide

You can (will) find detailed documentation [here](docs/DOCS.md). 
Note that these docs are currently in progress. 

> :warning: 1.7.0 Upgrade
> The 1.7.0 upgrade comes with big changes due to the update to the 5.0.0 version of the fabric8
> Kubernetes client. While this should improve the user experience quite nicely, there are a couple
> of things to be aware of when upgrading from a previous version as detailed below.

##### Overview of the 1.7.0 changes

- `Doneable` classes have been removed along with all the involved complexity
- `Controller` annotation has been simplified: the `crdName` field has been removed as that value is
  computed from the associated custom resource implementation
- Custom Resource implementation classes now need to be annotated with `Group` and `Version`
  annotations so that they can be identified properly. Optionally, they can also be annotated with
  `Kind` (if the name of the implementation class doesn't match the desired kind) and `Plural` if
  the plural version cannot be automatically computed (or the default computed version doesn't match
  your expectations).
- The `CustomResource` class that needs to be extended is now parameterized with spec and status
  types, so you can have an empty default implementation that does what you'd expect. If you don't
  need a status, using `Void` for the associated type should work.
- Custom Resources that are namespace-scoped need to implement the `Namespaced` interface so that
  the client can generate the proper URLs. This means, in particular, that `CustomResource`
  implementations that do **not** implement `Namespaced` are considered cluster-scoped. As a
  consequence, the `isClusterScoped` method/field has been removed from the appropriate
  classes (`Controller` annotation, in particular) as this is now inferred from the `CustomResource`
  type associated with your `Controller`.

Many of these changes might not be immediately apparent but will result in `404` errors when
connecting to the cluster. Please check that the Custom Resource implementations are properly
annotated and that the value corresponds to your CRD manifest. If the namespace appear to be missing
in your request URL, don't forget that namespace-scoped Custom Resources need to implement
the `Namescaped` interface.

#### Usage

We have several sample Operators under the [samples](samples) directory:
* *pure-java*: Minimal Operator implementation which only parses the Custom Resource and prints to stdout.
Implemented with and without Spring Boot support. The two samples share the common module.
* *spring-boot-plain/auto-config*: Samples showing integration with Spring Boot.
* *quarkus*: Minimal application showing automatic configuration / injection of Operator / Controllers.

And there are more samples in the standalone [samples repo](https://github.com/java-operator-sdk/samples):
* *webserver*: Simple example creating an NGINX webserver from a Custom Resource containing HTML code.
* *mysql-schema*: Operator managing schemas in a MySQL database.
* *tomcat*: Operator with two controllers, managing Tomcat instances and Webapps for these.

Add [dependency](https://search.maven.org/search?q=a:operator-framework) to your project with Maven:

```xml
<dependency>
  <groupId>io.javaoperatorsdk</groupId>
  <artifactId>operator-framework</artifactId>
  <version>{see https://search.maven.org/search?q=a:operator-framework for latest version}</version>
</dependency>
```

Or alternatively with Gradle, which also requires declaring the SDK as an annotation processor to
generate the mappings between controllers and custom resource classes:

```groovy
dependencies {
    implementation "io.javaoperatorsdk:operator-framework:${javaOperatorVersion}"
    annotationProcessor "io.javaoperatorsdk:operator-framework:${javaOperatorVersion}"
}
```

Once you've added the dependency, define a main method initializing the Operator and registering a controller.

```java
public class Runner {

   public static void main(String[] args) {
       Operator operator = new Operator(new DefaultKubernetesClient(),
           DefaultConfigurationService.instance());
       operator.register(new WebServerController());
   }
}
```

The Controller implements the business logic and describes all the classes needed to handle the CRD.

```java
@Controller
public class WebServerController implements ResourceController<WebServer> {

    @Override
    public DeleteControl deleteResource(CustomService resource, Context<WebServer> context) {
        // ... your logic ...
        return DeleteControl.DEFAULT_DELETE;
    }
    
    // Return the changed resource, so it gets updated. See javadoc for details.
    @Override
    public UpdateControl<CustomService> createOrUpdateResource(CustomService resource, Context<WebServer> context) {
        // ... your logic ...
        return UpdateControl.updateStatusSubResource(resource);
    }
}
```

A sample custom resource POJO representation

```java
@Group("sample.javaoperatorsdk")
@Version("v1")
public class WebServer extends CustomResource<WebServerSpec, WebServerStatus> implements Namespaced {}

public class WebServerSpec {

    private String html;

    public String getHtml() {
        return html;
    }

    public void setHtml(String html) {
        this.html = html;
    }
}
```
          
#### Quarkus

A [Quarkus](https://quarkus.io) extension is also provided to ease the development of Quarkus-based operators.

Add [this dependency](https://search.maven.org/search?q=a:operator-framework-quarkus-extension)
to your project:

```xml
<dependency>
 <groupId>io.javaoperatorsdk</groupId>
 <artifactId>operator-framework-quarkus-extension</artifactId>
 <version>{see https://search.maven.org/search?q=a:operator-framework-quarkus-extension for latest version}</version>
</dependency>
```

Create an Application, Quarkus will automatically create and inject a `KubernetesClient`, `Operator` 
and `ConfigurationService` instances that your application can use, as shown below:

```java
@QuarkusMain
public class QuarkusOperator implements QuarkusApplication {

  @Inject KubernetesClient client;

  @Inject Operator operator;

  @Inject ConfigurationService configuration;

  public static void main(String... args) {
    Quarkus.run(QuarkusOperator.class, args);
  }

  @Override
  public int run(String... args) throws Exception {
    final var config = configuration.getConfigurationFor(new CustomServiceController(client));
    System.out.println("CR class: " + config.getCustomResourceClass());

    Quarkus.waitForExit();
    return 0;
  }
}
```

#### Spring Boot

You can also let Spring Boot wire your application together and automatically register the controllers.

Add [this dependency](https://search.maven.org/search?q=a:operator-framework-spring-boot-starter) to your project:

```xml
<dependency>
 <groupId>io.javaoperatorsdk</groupId>
 <artifactId>operator-framework-spring-boot-starter</artifactId>
 <version>{see https://search.maven.org/search?q=a:operator-framework-spring-boot-starter for latest version}</version>
</dependency>
```

Create an Application
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

#### Spring Boot test support

Adding the following dependency would let you mock the operator for the 
tests where loading the spring container is necessary, 
but it doesn't need real access to a Kubernetes cluster.

```xml
<dependency>
 <groupId>io.javaoperatorsdk</groupId>
 <artifactId>operator-framework-spring-boot-starter-test</artifactId>
 <version>{see https://search.maven.org/search?q=a:operator-framework-spring-boot-starter for latest version}</version>
</dependency>
``` 

Mock the operator:
```java
@SpringBootTest
@EnableMockOperator
public class SpringBootStarterSampleApplicationTest {

  @Test
  void contextLoads() {}
}
```
