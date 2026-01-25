> What is the Spring, Spring Framework, or Spring Boot? Why does it exist and how does it work?

### Spring
Just the brand under which: 
- People work on the [[#Spring Framework]]
- All the tools associated with Spring Framework are covered
- All projects that evolved from Spring Framework are covered 

### Spring Framework
A Java enterprise development framework that, in simple terms, promises to help:
1. Simplify the *lifecycle* management of complex Java components
2. Externalize the *configuration* of complex Java components

Back in 2004 it was adopted with a huge success because Java developers were pressured to adopt very heavyweight and very complicated solutions like **J2EE** (Java 2 Platform, Enterprise Edition; now known as Jakarta EE - formerly Java EE) or **EJBS** (Jakarta Enterprise Beans). Spring Framework showed that it can be done a lot simpler. 

At its core, Spring Framework is really **just a dependency injection container**, with a couple of convenience layers added on top (think of: database access, proxies, AOP, RPC, a web mvc framework).

Spring Framework is the basis for all other projects that exist nowadays in the Spring ecosystem like:
- Spring Data
- Spring JPA
- Spring Boot
- Spring Web Service
- Spring AI
- Spring Web MVC
- Spring for Kafka
- and many more...

The key benefits to using Spring Framework:
- **[[#Dependency Injection & Inversion of Control|Dependency Injection]] and loose coupling** - objects don't create their dependencies, easy to swap implementations, improved testability, much less boilerplate code

- **Modularity and flexibility** - only needed modules are used, framework can be used in variety of context ranging from standalone applications to distributed systems.

- **Testing** - easy mocking, lightweight testing

- **Efficient development** - allows to focus on business values straight away rather than configuration, dependencies, boilerplate code - infrastructure is handled by spring

- **POJO-based development** - no need to extend framework classes, no forced interfaces, minimal framework lock-in

- **Externalized configuration** - no hard-coded configuration

- **Consistent programming model** - across all spring modules and projects

- **Community** - spring is an open-source project with a large and active community providing a ton of resources and support for developers

#### Dependency Injection & Inversion of Control
> It's a design pattern where dependencies are supplied to objects at runtime instead of the objects creating them themselves.

Without Dependency Injection, the class has to take care of creating the dependency object. This poses some problems:
- Tight coupling - it's hard to swap implementations
- Complicated testing - no easy way to mock objects
- A lot of boilerplate code - many `new` occurrences

So instead *"I will create my own dependencies"* the class says *"I need these dependencies"* and Spring Framework handles the rest. This is known also as the *Hollywood Principle* - don't call us, we will call you.

The dependency injection in return gives:
- Loose coupling
- Easier unit testing - mock dependencies
- Easier configuration changes
- Cleaner, more maintainable code

DI is the technique that implements the principle called ***Inversion of Control (IoC)***. Meaning:
> The control of creating objects and managing their lifecycle if inverted from the business logic to the external provider (in Spring it is the [[#IoC Container]])

IoC is also known as the *Hollywood Principle* - don't call us, we will call you (the framework calls me rather me calling the framework).
- My class creating its own dependencies = me calling to Hollywood
- My class instantiated along with its dependencies by framework = Hollywood calling me
##### Types of DI
*TODO*

#### IoC Container
> It's the engine that makes Dependency Injection possible. It's a part of the framework which manages instantiating, configuring, and assembling beans (a.k.a objects).

In theory it's a design pattern where the management of object creation and their lifecycle is transferred from the business logic to the framework. More specifically it is responsible for:
- Object creation - also called [[#Beans]]
- Dependency configuration via DI
- Management the lifecycle of beans
- Reading configuration metadata

The instructions on how to create, configure, and assemble are read from *configuration metadata*. The configuration metadata can be represented as:
- Annotated component class
- Configuration class with factory methods
- External XML file(s)
- Groovy script

In practice, the application classes are combined with configuration metadata so that, after the IoC container is created and initialized, you have a fully configured and executable system or application (in other words, *application classes are created with necessary dependencies*).
![[Pasted image 20260125150023.png]]
<sup>How IoC container works in Spring</sup>

Two types of containers are provided in Spring:
1. `BeanFactory` container 
	- The basic container providing support for DI
	- The beans are instantiated only when explicitly requested
	- Lightweight and suitable for resource-constrained environments
2. `ApplicationContext` container - 
	- An advanced container built on top of `BeanFactory`
	- Adds an extra features such as internationalization, event propagation, integration with other Spring modules, etc.
	- The beans are instantiated and configured on startup


##### Beans
> Objects managed under the IoC container. The result of an application class + configuration metadata.

Within container the beans are represented as `BeanDefinition` which is essentially a recipe for creating one or more objects. A `BeanDefinition` contains:
- A package-qualified class name - typically the actual implementation class of the bean
- Behavioral configuration elements - how should bean behave in the container (scope, lifecycle callback, etc.)
- References to other beans - dependencies
- Other configuration settings - e.g. the max number of connections for bean managing connection pool

When the container is asked to provide a bean, it looks at the `BeanDefinition` of the named bean and uses the configuration metadata encapsulated in the bean definition to create (or acquire) an actual object.

The beans are created via class [[Java#Annotations|annotations]] provided by Spring, such as `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`, etc. 

Another way of creating beans is via configuration class annotated with `@Configuration`, by using annotation `@Bean` on methods returning an instance of a class that you want Spring to manage.

###### Scopes
A way of telling a container how many instances of the bean should be created:
- **Singleton** - for every dependency, exactly one instance will be created and shared
- **Prototype** - for each dependency, a new instance will be created
- **Session** - for each user HTTP session, a new instance will be created
- and more...