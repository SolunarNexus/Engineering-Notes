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
> It's the engine that makes Dependency Injection possible. It's a part of the framework which manages objects' lifecycle (dependencies a.k.a beans), and their dependencies.

In theory it's a design pattern where the management of object creation and their lifecycle is transferred from the business logic to the framework. More specifically it is responsible for:
- Object creation - also called [[#Beans]]
- Dependency configuration via DI
- Management the lifecycle of beans
- Reading configuration metadata

##### Beans