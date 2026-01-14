> *Mainly notes about internal details of how Java does the magic.*

### Basics
#### Class
A template/blueprint for object creation. Represents a class of objects that share the same characteristics a.k.a are of the same kind.
```java
class Bicycle {
    int cadence = 0;
    int speed = 0;
    int gear = 1;

    void changeCadence(int newValue) {
         cadence = newValue;
    }

    void changeGear(int newValue) {
         gear = newValue;
    }

    void speedUp(int increment) {
         speed = speed + increment;   
    }

    void applyBrakes(int decrement) {
         speed = speed - decrement;
    }
}
```
More at https://dev.java/learn/classes-objects/

#### Object
An instance of a class. In software, it is a bundle of related state and behavior. For instance, a dog has the state (name, color, breed) and behavior (barking, fetching, sleeping). An object stores its state in *fields* and exposes its behavior through *methods* (or functions in functional languages). Methods operate on object's internal state and serve as the primary mechanism for object-object communication.

More at https://dev.java/learn/classes-objects/

#### Interface
A group of declarations of related functionality/methods a class promises to provide. Defines a contract between a class and the outside world (e.g. caller) and this contract is enforced at the build time by compiler. 

Interface is a reference type just like a [[#Class]] (which means interface can be used as a type for objects whose class implements that interface), that can contain only:
- constants
- method signatures (abstract method)
- default methods (with default modifier) (can define method's body)
- static methods (private or public, **not** protected) (with static modifier) (can define method's body)
- instance non-abstract methods (**only** private) (can define method's body)
- and nested types
Interfaces can only be implemented by other classes **or** extended by other interfaces. All method signatures are implicitly `public` in interfaces, so you can omit the `public` modifier. All constant values are implicitly `public static final` - you can omit these modifiers.

```java
public interface OperateCar extends InterfaceA, InterfaceB, InterfaceC {

    // constant declarations (implicitly public static final)
    double E = 2.718282;
 
	// nested type - enum (implicitly public)
	private enum Direction {
		LEFT, RIGHT
	}
	
	// abstract method (implicitly public)
	void signalTurn(Direction direction, boolean signalOn);
	
	// default method (implicitly public)
	default void startEngine(){
		/* implementation */
	}
	
	// instance non-abstract method (from java 9)
	// (implicitly private and cannot be abstract nor default)
	private void reset() {
		/* implementation */
	}
	
	// static method (implicitly public)
	static void printStatus(){
		/* optional implementation */
	}
}
```

The `public` access specifier indicates that the interface can be used by any class in any package. Without it, the interface can be used only by classes defined in the same package as the interface.

The class implementing an interface must provide a method body for each method declared in that interface (excluding default methods - those can be overriden if one wishes so). The class can implement multiple interfaces.

```java
public class OperateBMW760i implements OperateCar {

    // the OperateCar method signatures, with implementation --
    // for example:
    public int signalTurn(Direction direction, boolean signalOn) {
       // code to turn BMW's LEFT turn indicator lights on
       // code to turn BMW's LEFT turn indicator lights off
       // code to turn BMW's RIGHT turn indicator lights on
       // code to turn BMW's RIGHT turn indicator lights off
    }

    // other members, as needed -- for example, helper classes not 
    // visible to clients of the interface
}
```

More at https://dev.java/learn/interfaces/

#### Package
A namespace that organizes a set of related classes and interfaces (think of folders in a PC). To make types easier to find and use, to avoid naming conflicts, and to control access, programmers bundle groups of related types into packages. Benefits:
- you and other programmers can easily determine that types in the same package are related
- you and other programmers can easily find the types
- the names of types will not conflict with the type names in other packages
- you can allow types within the package to have unrestricted access to one another yet still restrict access for types outside the package 
Every file in Java **must** have specified package name. 

More at https://dev.java/learn/packages/

---
### JDK vs JRE vs JVM
#### JRE - Java Runtime Environment
Contains everything needed to **run** any java program:
- [[#Java Standard Library]]
- [[#JVM - Java Virtual Machine]]
JVM would take the java program, link all the classes and dependencies from the Java Standard library that it needs to execute the program and run.
#### Java Standard Library
The set of helper classes and files to enable java do what it has to do. It contains the main java packages such as:
- *java.lang.\** - String, Math, Exception, Throwable, Object (root of the all other java classes)
- *java.util.\** - List, Map, Set, Collection
- *java.io.\** - input and output
- *java.sql.\** - interaction with databases
#### JVM - Java Virtual Machine
In simplicity, takes responsibility for executing the java program. It loads up the byte code of all .class files (compiled .java files) into memory and executes the program. Behind the scenes takes care of all the stuff to run your program smoothly and you don't have to be aware of it e.g.:
- Ensures that the program runs smoothly on whatever platform without your co-operation
- Handles memory and [[#Garbage Collector|garbage collection]] automatically
- Handles multi-threading (if the program uses it)
- Built-in security checks
#### JDK - Java Development Kit
Enables the development of java programs. Contains everything needed for creation, modification, compilation, debugging, testing, and running java programs:
- [[#JRE - Java Runtime Environment]] for executing java programs
- The java compiler - *javac*
- Java Library source code (in addition to binaries in [[#JRE - Java Runtime Environment|JRE]])
- Debugger
- Monitoring tools - e.g. *jconsole*
- Tools for creating and managing JAR files
- ...and many more tools for creation, testing, using and more (a lot)
![[Pasted image 20260111175409.png]]
### Garbage Collector
Basically manages the memory of the program behind the scenes at runtime (and you don't have to). It removes the objects from memory that are not used anymore. The object is marked for removal if there are **no** references pointing to that instance.

The default GC initially stores all the instances in the so-called Young Generation heap. After some time the GC inspects the Young Generation heap - *mark and sweep* action. *Mark and sweep* action consists of 
- marking objects in the heap that are still in use
- and removing (sweeping) all the other objects
GC also optimizes this process by observing the lifetime of objects. If some objects are still in use after some time, the GC assumes they are important for the run of the program and will still be in use in the future. Such objects are moved to the so-called Old Generation heap. The GC then does *mark and sweep* on Old Generation heap way less often and the number of objects participating in Young Generation heap decreases which makes garbage collection more efficient.

---
### String Immutability
Refers to immutability of String objects in a memory. The benefits are:
- **Saves memory** - upon creation of a string variable, the string object is stored in the "string pool" in memory. All subsequent string variables with the same value point to the same object is the string pool. With mutable strings this wouldn't be possible (the changes in the string object would reflect in all instances pointing to it)
- **Thread safety** - threads can safely access the string objects without the risk of introducing inconsistent state or unexpected behavior
- **Security** - prevents change of referenced string object which might pose a security risk

---
### Pass-by-value
Java always passes objects by value BUT that value is the reference (kinda pass-by-reference). It is possible to modify the internal state of the object, but it is impossible to change reference to another object - the reference to the original object may be changed but it would be available only inside the current scope (method's body).

---
### Exceptions
*An exception is an event, which occurs during the execution of a program, that disrupts the normal flow of the program's instructions.*

When an error occurs within a method, the method creates an exception object containing information about the error, its type and the state of the program when the error occurred. This object is than handed off to the runtime system. Creating an exception object and handing it to the runtime system is called **throwing an exception**.

![[Pasted image 20260114135507.png]]
<sup>The call stack</sup>
After throwing an exception, the system tries to find someone/something to handle the exception. The runtime system searches the *call stack* (i.e. the list of methods that had been called to get to the method where the error occurred) for a method that contains a block of code that can handle the exception. A block of code handling the exception is called an *exception handler*.

When the appropriate exception handler is found, the runtime system passes the exception object to the handler. This is called *catching the exception*. If the system cannot find any appropriate exception handler, the thread in which the exception occurred, terminates. If that thread is the main thread, the runtime system terminates as well (and consequently the program). 

![[Pasted image 20260114141444.png]]
<sup>Searching the call stack for the exception handler</sup>

#### Checked Exception
Represents an exceptional condition that application should anticipate and recover from (e.g. supplying a path to non-existent file in the constructor of `java.io.FileReader`). These exceptions are checked for at compile time. The code that might throw checked exception must honor the *Catch of Specify Requirement* which means that the code must do either of:
1. Enclose the method call with a try-catch block
2. Provide the *throws* clause in the method signature that lists the exception(s) - passes the responsibility to deal with the exception to a caller
Either way, the caller **must** deal with the exception. All exceptions are checked exceptions except `Error`, `RunTimeException` and their subclasses.

#### Unchecked Exception
All exceptions of type `RuntimeException`or `Error` and their subclasses. Represents exceptional conditions that are external (`Error`) or internal (`RuntimeException`) to the application, and that the application usually cannot anticipate or recover from. The `Error` can be caused by system or hardware malfunction external to the application. The `RuntimeException` usually indicate programming bugs internal to the application. The caller does not have to deal with the exception at compile time (but should prepare for such cases).

#### Handling Exceptions
Exceptions are handled using try-catch-finally block of code where:
- `try` block encapsulates the code that might throw exceptions
- `catch` block is an exception handler that handles the type of exception specified in the argument
- `finally` block contains the code that is executed always regardless what happens in try/catch block
The `finally` block is a key tool for preventing resource leaks and closing a file or other resource that must be closed after the work is done. An alternative to `finally` is using *try-with-resources* statement which automatically releases system resources when no longer needed. 
```java
// use of try-with-resources statement
static String readFirstLineFromFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}
```

More at  https://dev.java/learn/exceptions/

### Method .equals() vs ==
### Map implementation
### HashTable implementation
### Multi-threading