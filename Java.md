> *Notes about internal details of how Java does the magic.*
> 
> Concepts, internal working, and interview questions: https://javaconceptoftheday.com/
> Official Java tutorial with focus on syntax: https://dev.java/learn/

1. [[#Core Concepts|Core Concepts]]
	1. [[#Core Concepts#Program Lifecycle|Program Lifecycle]]
	2. [[#Core Concepts#JDK vs JRE vs JVM|JDK vs JRE vs JVM]]
	3. [[#Core Concepts#Garbage Collector|Garbage Collector]]
	4. [[#Core Concepts#Interfaces|Interfaces]]
	5. [[#Core Concepts#Packages|Packages]]
	6. [[#Core Concepts#Contract of equals() and hashCode()|Contract of equals() and hashCode()]]
	7. [[#Core Concepts#Object Immutability|Object Immutability]]
	8. [[#Core Concepts#Parameter Passing Mechanism|Parameter Passing Mechanism]]
2. [[#Exceptions|Exceptions]]
	1. [[#Exceptions#Advantages|Advantages]]
	2. [[#Exceptions#Checked Exception|Checked Exception]]
	3. [[#Exceptions#Unchecked Exception|Unchecked Exception]]
	4. [[#Exceptions#Handling Exceptions|Handling Exceptions]]
	5. [[#Exceptions#Creating custom Exceptions|Creating custom Exceptions]]
3. [[#Collections|Collections]]
	1. [[#Collections#Benefits over traditional data structures|Benefits over traditional data structures]]
	2. [[#Collections#Iterables|Iterables]]
	3. [[#Collections#Hashmap|Hashmap]]
4. [[#Concurrency|Concurrency]]
	1. [[#Concurrency#Threads|Threads]]
	2. [[#Concurrency#Synchronization|Synchronization]]
	3. [[#Concurrency#Virtual Threads|Virtual Threads]]

### Core Concepts
#### Program Lifecycle
After editing the program in e.g. text editor or an IDE, the program is then compiled into a *bytecode* with the Java's compiler `javac`. The bytecode is machine/platform independent code that is runnable at every machine that has installed [[#JRE - Java Runtime Environment|JRE]]. The output of the compiler is the *.class* which encapsulates the byte code.

The program is executed by [[#JVM - Java Virtual Machine|JVM]] which loads the bytecode into the memory where the interpreter reads the bytecode instruction by instruction and the JIT (Just-In-Time) compiler translates frequently executed bytecode into a native machine code for better performance. 
#### JDK vs JRE vs JVM
##### JRE - Java Runtime Environment
Contains everything needed to **run** any java program:
- [[#Java Standard Library]]
- [[#JVM - Java Virtual Machine]]
JVM would take the java program, link all the classes and dependencies from the Java Standard library that it needs to execute the program and run.

##### Java Standard Library
The set of helper classes and files to enable java do what it has to do. It contains the main java packages such as:
- *java.lang.\** - String, Math, Exception, Throwable, Object (root of the all other java classes)
- *java.util.\** - List, Map, Set, Collection
- *java.io.\** - input and output
- *java.sql.\** - interaction with databases

##### JVM - Java Virtual Machine
In simplicity, takes responsibility for executing the java program. It loads up the byte code of all .class files (compiled .java files) into memory and executes the program. Behind the scenes takes care of all the stuff to run your program smoothly and you don't have to be aware of it e.g.:
- Ensures that the program runs smoothly on whatever platform without your co-operation
- Handles memory and [[#Garbage Collector|garbage collection]] automatically
- Handles multi-threading (if the program uses it)
- Built-in security checks

##### JDK - Java Development Kit
Enables the development of java programs. Contains everything needed for creation, modification, compilation, debugging, testing, and running java programs:
- [[#JRE - Java Runtime Environment]] for executing java programs
- The java compiler - *javac*
- Java Library source code (in addition to binaries in [[#JRE - Java Runtime Environment|JRE]])
- Debugger
- Monitoring tools - e.g. *jconsole*
- Tools for creating and managing JAR files
- ...and many more tools for creation, testing, using and more (a lot)
![[Pasted image 20260111175409.png]]

---
#### Garbage Collector
Basically manages the memory of the program behind the scenes at runtime (and you don't have to). It removes the objects from memory that are not used anymore. The object is marked for removal if there are **no** references pointing to that instance.

The default GC initially stores all the instances in the so-called Young Generation heap. After some time the GC inspects the Young Generation heap - *mark and sweep* action. *Mark and sweep* action consists of 
- marking objects in the heap that are still in use
- and removing (sweeping) all the other objects
GC also optimizes this process by observing the lifetime of objects. If some objects are still in use after some time, the GC assumes they are important for the run of the program and will still be in use in the future. Such objects are moved to the so-called Old Generation heap. The GC then does *mark and sweep* on Old Generation heap way less often and the number of objects participating in Young Generation heap decreases which makes garbage collection more efficient.

---
#### Interfaces
An interface is a group of declarations of related functionality/methods a class promises to provide. Defines a contract between a class and the outside world (e.g. caller) and this contract is enforced at the build time by compiler. 

Interface is a reference type just like a class (which means interface can be used as a type for objects whose class implements that interface), that can contain only:
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

---
#### Packages
A package is a namespace that organizes a set of related classes and interfaces (think of folders in a PC). To make types easier to find and use, to avoid naming conflicts, and to control access, programmers bundle groups of related types into packages. 

Benefits:
- you and other programmers can easily determine that types in the same package are related
- you and other programmers can easily find the types
- the names of types will not conflict with the type names in other packages
- you can allow types within the package to have unrestricted access to one another yet still restrict access for types outside the package

Every file in Java **must** have specified package name. 

More at https://dev.java/learn/packages/

---
#### Contract of equals() and hashCode()
>**TLDR;** `equals()` compares inner state of objects, `hashCode()` provides unique integer identifier for objects and is especially useful with hash table data structures (`HashMap`), `==` operator compares references of two objects.

By default, the `Object` class implements both methods. Providing a custom implementation of these methods is not always needed. For entity classes and **objects having intrinsic identity**, the default implementation suffices. But for **value objects** it is better to base equality on their properties - providing custom implementation of `equals()` and `hashCode()`. 

Custom implementation typically is not written by hand, one can:
- Generate methods using IDE. 
- Use helper classes of [Apache Commons Lang](https://www.baeldung.com/java-commons-lang-3) and [Google Guava](https://www.baeldung.com/whats-new-in-guava-19) to simplify writing these methods.
- Use annotation `@EqualsAndHashCode` from [Project Lombok](https://www.baeldung.com/intro-to-project-lombok)
##### equals()
By default, the implementation of `equals()` method compares object identities, related to `==` operator comparing object references. To change this behavior in custom classes, we must override this method in order to compare the state (field values) of objects as well.

Java SE defines a contract that the custom implementation of the `equals()` method must fulfill:
- **reflexive** - an object must equal itself.
- **symmetric** - `a.equals(b)` must return the same result as `b.equals(a)`.
- **transitive** - if `a.equals(b)` and `b.equals(c),` then also `a.equals(c)`.
- **consistent** - the value of `equals()` should change only if a property that is contained in `equals()` changes (no randomness allowed).

Example:
```java
// "Money" is a custom class

@Override
public boolean equals(Object o) {
	// 1. Objects are equal if they share the same reference (identity)
    if (o == this)
        return true;
    // 2. Objects need to be of the same instance class in order to be equal
    if (!(o instanceof Money))
        return false;
    Money other = (Money)o;
    // 3. Objects are equal if their attributes are equal as well
    boolean currencyEquals = 
		(this.currency == null && other.currency == null) ||
		(this.currency != null && this.currency.equals(other.currency));
		
    return this.amount == other.amount && currencyEquals;
}
```

The contract of `equals()` can be broken via inheritance when extending a class that has overriden the `equals()` method as well. To avoid such mistakes, it is better to favor composition over inheritance.

Example:
```java
class A {
	private int num;
	
	@Override
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof A))
            return false;
        A other = (A)o;
        
        return this.num == other.num; 
    }
}

class B extends A {
	private String word;
	
	@Override
	public boolean equals(Object o){
		if (o == this)
            return true;
        if (!(o instanceof B))
            return false;
        B other = (B)o;

		boolean wordEquals = 
			(this.word == null && other.word == null) || 
			(this.word != null && this.word.equals(other.word));

        return this.num == other.num && wordEquals; 
	}
}

A first = new A(42);
B second = new B(42, "word");

// As expected. Class A is not an instance of class B
second.equals(first) => false 
// Wrong. Ignoring the new attribute "word". Breaks the symmetry.
first.equals(second) => true
```

##### hashCode()
Now the `hashCode()` method returns an integer identifying the current instance of the class. The default implementation usually converts the internal address of the object into an integer, but it is JVM-specific.

Java SE specifies a contract for `hashCode()` as well:
- **internal consistency** - the value of `hashCode()` may only change if a property acting in `equals()` changes.
- **equals consistency** - objects that are equal to each other must return the same hash code.
- **collisions** - unequal objects may have the same hashCode.

Because of the second criterion, **if we override the `equals()` we must override `hashCode()` and vice versa**. 

---
#### Object Immutability
An immutable object has **an internal state that remains constant after object's creation.** Meaning that an immutable object guarantees that it will behave in the same way during its whole lifetime.

To make an object immutable **we must guarantee its internal state won't change** during its lifetime:
```java
// Final class cannot be extended
public final class Money {
	// All primitive data types won't change with final; that's guaranteed
    private final double amount;
    // With higher-level objects we must rely on their immutability. To maintain
    // object's immutability event with mutable fields (e.g. Collections or
    // Arrays), return defensive copies.
    private final Currency currency;

	// Constructor is the only place allowed to set the internal state
	public Money(double amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }

	// Only read-only methods can be exposed
	public Currency getCurrency() {
        return currency;
    }

    public double getAmount() {
        return amount;
    }
}
```

The general benefits are:
- **Thread safety** - ability to share safely such objects among multiple threads without the risk of introducing inconsistent state or unexpected behavior
- **No side effects** - none of the objects referencing immutable object will notice any difference and that is a plus during debugging (you don't have to inspect where the value changed)
- **Security** - inner state cannot be altered maliciously after creation
- **Caching** - ideal for caching mechanisms as their state does not change

On top of that, Strings in Java are immutable with additional benefit:
- **Saves memory** - upon creation of a string variable, the string object is stored in the "string pool" in memory. All subsequent string variables with the same value point to the same object in the string pool. With mutable strings this wouldn't be possible (the changes in the string object would reflect in all instances pointing to it)

---
#### Parameter Passing Mechanism
Java is a **"pass-by-value"** language always passing arguments by value. 

In case of **primitive values**, the value is **simply copied** inside stack memory which is then passed to the method.  

In case of **non-primitive** higher-level objects, the **reference** in the stack memory pointing to the data in the heap **is copied** and passed to the method. Very similar to the "pass-by-reference" mechanism but with a distinction:
- it is **possible** to modify the internal state of the object and that change reflects outside the scope
- it is **impossible** to assign another object to the same reference - the local variable can be reassigned but the change would take effect only inside the current scope (method's body).

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

#### Advantages
1. **Separation of error-handling code from "Regular" code** - solves problems of traditional error-management techniques (returning error code), where the actual logic can be lost in a spaghetti code of error detection, handling, and reporting. Consider
```java
errorCodeType readFile {
	initialize errorCode = 0;
    
    // open the file;
    if (theFileIsOpen) {
        // determine the length of the file;
        if (gotTheFileLength) {
            // allocate that much memory;
            if (gotEnoughMemory) {
                // read the file into memory;
                if (readFailed) {
                    errorCode = -1;
                }
            } else {
                errorCode = -2;
            }
        } else {
            errorCode = -3;
        }
        // close the file;
        if (theFileDidntClose && errorCode == 0) {
            errorCode = -4;
        } else {
            errorCode = errorCode and -4;
        }
    } else {
        errorCode = -5;
    }
    return errorCode;
}
```
and the same code but with the use of exceptions:
```java
readFile {
    try {
        // open the file;
        // determine its size;
        // allocate that much memory;
        // read the file into memory;
        // close the file;
    } catch (fileOpenFailed) {
       // doSomething;
    } catch (sizeDeterminationFailed) {
       // doSomething;
    } catch (memoryAllocationFailed) {
       // doSomething;
    } catch (readFailed) {
       // doSomething;
    } catch (fileCloseFailed) {
       // doSomething;
    }
}
```
2. **Propagation errors up the call stack** - only methods that are interested in errors have to worry about detecting errors, all subsequent methods between the point of throwing exception and the method that cares about that exception don't have to do nothing except specifying that they may throw an exception using `throws` clause.
3. **Grouping and differentiating error types** - because all exceptions are objects, the grouping and categorizing is a natural outcome of the class hierarchy. This way we are able to catch the whole hierarchy of exceptions (e.g. `IOException`) 

#### Checked Exception
Represents an exceptional condition that application should anticipate and recover from (e.g. supplying a path to non-existent file in the constructor of `java.io.FileReader`). These exceptions are checked for at compile time. The code that might throw checked exception must honor the *Catch of Specify Requirement* which means that the code must do either of:
1. Enclose the method call with a try-catch block
2. Provide the *throws* clause in the method signature that lists the exception(s) - passes the responsibility to deal with the exception to a caller
Either way, the caller **must** deal with the exception. All exceptions are checked exceptions except `Error`, `RunTimeException` and their subclasses.

#### Unchecked Exception
All exceptions of type `RuntimeException`or `Error` and their subclasses. Represents exceptional conditions that are external (`Error`) or internal (`RuntimeException`) to the application, and that the application usually cannot anticipate or recover from. 

The `Error` can be caused by system or hardware malfunction external to the application and that usually prevent the JVM from running.

The `RuntimeException` usually indicate programming bugs internal to the application. The caller does not have to deal with the exception at compile time (but should prepare for such cases).

![[Pasted image 20260114175851.png]]
<sup>The Throwable hierarchy</sup>
#### Handling Exceptions
Exceptions are handled using try-catch-finally block of code where:
- `try` block encapsulates the code that might throw exceptions
- `catch` block is an exception handler that handles the type of exception specified in the argument
- `finally` block contains the code that is executed always regardless what happens in try/catch block

General example:
```java
public void writeList() {
    PrintWriter out = null;

    try {
        out = new PrintWriter(new FileWriter("OutFile.txt"));
        
        for (int i = 0; i < SIZE; i++) {
            out.println("Value at: " + i + " = " + list.get(i));
        }
    } catch (IndexOutOfBoundsException e) {
        System.err.println("IndexOutOfBoundsException: " +  e.getMessage());
    } catch (IOException e) {
        System.err.println("Caught IOException: " +  e.getMessage());
    } finally {
        if (out != null) out.close();
    }
}
```

The `finally` block is a key tool for preventing resource leaks and clean-up e.g. closing a file or other resource when no longer needed. An alternative to `finally` is using *try-with-resources* statement which automatically releases system resources after exiting the block.
```java
static String readFirstLineFromFile(String path) throws IOException {
    // use of try-with-resources statement
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}
```

Note that in case of exception, the runtime system inspects the `catch` clauses in a top-down manner. The clause whose argument's type matches the thrown exception type **first** is selected for handling the exception. Therefore, the `catch` clauses should be ordered from the most specialized to the most generic. For example if `Exception e` was the argument of the first `catch` clause, such code would end with error saying that the exception was already caught. Consider:
```java
try {
    // some IO code
} catch (IOException ioe) {
    // IOException is a supertype of FileNotFoundException - compiler error
} catch (FileNotFoundException fnfe) {

}
```

#### Creating custom Exceptions
The general rule: *"If a client can reasonably be expected to recover from an exception, make it a checked exception. If a client cannot do anything to recover from the exception, make it an unchecked exception."*

More at  https://dev.java/learn/exceptions/
 
---
### Collections
Represent a set if interfaces that model different ways of storing data in different data structures. For each interface, at least one implementation is provided. Choosing the right implementation (and interface) depends on what is your intention. 

#### Benefits over traditional data structures
1. A collection tracks the number of elements it stores
2. The capacity is not limited (the only limitation is available memory)
3. Control over what elements are stored in a collection - e.g. prevent null values
4. Ability to be queried for the presence of a given element
5. Operations for intersecting or merging with another collection
6. And more...

#### Iterables
The main category of collections are those implementing the *Iterable* interface directly or indirectly. You are able to iterate over an object implementing this interface. 

![[Pasted image 20260115130537.png]]
<sup>The collection interface hierarchy</sup>
![[Pasted image 20260118153409.png]]<sup>Closeup on collection hierarchy</sup>

#### Hashmap
The classic data structures for storing key-value pairs. Map interface hierarchy is separate from collections - collection represents a group of elements (values only) i.e. they are conceptually different.
![[Pasted image 20260118153600.png]]
<sup>The map interface hierarchy</sup>
![[Pasted image 20260118153630.png]]
<sup>Closeup on map hierarchy</sup>
The key of an entry serves for identification of a value. The hashcode of the key is used for that. The type of the key is extremely important - while it is possible to use a mutable key, it is dangerous as mutating it may lead to changing its hashcode value and its identity. This may make the entry unrecoverable or return a different entry when querying a map. In short: it can corrupt the map.

##### HashMap implementation details
Uses a hash table internally which is organized into an array of buckets. Initially, HashMap allocates 16 buckets in heap memory. Each bucket references a singly-linked list containing one or more entries. 
![[Pasted image 20260118162419.png]]
<sup>Example of HashMap</sup>
HashMap has also a so-called **form factor** i.e. how full the map can become before it automatically allocates more memory and rehashes its contents. By default the form factor is 75% i.e. after reaching 75% of capacity -> allocate more memory.

When a key-value pair is inserted, the hash code of the key is calculated using the `hashCode()` method, and then an index is derived to find the bucket (array position) using a **hash function** (e.g. modulo size of the map).

A collision can occur in case that two different keys map to the same bucket. Their hash code can differ, but hash function can place two objects in the same bucket. In case of a collision, HashMap does :
1. Check if both keys have the same hash code and are equal in accordance with `equals()` method. If yes then it is a duplicate and the old value is overwritten with a new one.
2. Otherwise, add the new key-value pair to the linked list
Upon retrieving, the HashMap:
- Calculates the hash code of a given key
- Calculates the bucket index from hash code (applying hash function)
- Goes to the bucket and iterate through the linked list until the element's key is not equal to the given key using `equals()` method.
![[Pasted image 20260118165728.png]]<sup>Collision in HashMap using Linked List</sup>

Starting from Java 8, when a number of nodes within a single bucket passes a threshold called *Treefiy Threshold* which is by default 8, the HashMap converts the internal structure of that bucket from a linked list to a balanced tree. This optimizes lookup efficiency from `O(n)` to `O(log n)`.The reverse operation applies when the size of HashMap shrinks under certain threshold (tree -> linked list).
![[Pasted image 20260118165649.png]]
<sup>Collision in HashMap using balanced Tree</sup>

##### HashMap vs TreeMap vs LinkedHashMap
All of them implement `Map` interface and offer mostly the same functionality. The most important difference is the order in which iteration through the entries will happen.
- `HashMap` makes absolutely no guarantees about the iteration order. It can (and will) even change completely after its size exceeds a certain threshold (then re-hashing occurs)
- `TreeMap` will iterate according to the "natural ordering" of the keys according to their `compareTo()` method (or an externally supplied `Comparator`). Additionally, it implements the [`SortedMap`](http://java.sun.com/javase/6/docs/api/java/util/SortedMap.html) interface, which contains methods that depend on this sort order.
- `LinkedHashMap` will iterate in the order in which the entries were put into the map

##### HashSet
Internally utilizes a [[#Hashmap]] to store its elements, which is created in HashSet's constructor. The elements contained in the HashSet act as keys of that HashMap instance. The value associated with those keys will be always a constant (e.g. `private static final Object PRESENT = new Object()`).

Possible implementation of HashSet's constructor:
```java
public class HashSet<E> {
	// Internal map instance
	private transient HashMap<E, Object> map;
	// Dummy value associated with every key in the map
	private static final Object PRESENT = new Object();
	
	public HashSet() {
	        map = new HashMap<>();
	}
	// other methods
}
```

Insertion can look like this:
```java
// ...
public boolean add(E e) {
	// 1. When inserting a duplicate, the value gets updated and the old
	// value is returned (in this case nothing gets updated as new == old)
	// 2. Inserting a unique value, would produce a null 
	return map.put(e, PRESENT) == null;
}
// ...
```

Removal is similar:
```java
public boolean remove(Object o) {
	// 1. When the key e is present in the map such that Object.equals(o, e)
	// then it is removed from the map and the associated value is returned
	// 2. If the key was absent, null is returned
	return map.remove(o) == PRESENT;
}
```

More at https://dev.java/learn/api/collections-framework/

---
### Concurrency
> List of questions with answers on various topics on concurrency: https://solutionsarchitecture.medium.com/java-concurrency-tutorial-from-basics-to-advanced-89f3f6d1a9b9

Involves managing multiple threads to perform tasks simultaneously which in turn improves performance and responsiveness of modern applications. Common concepts:
1. **Threads** - independent units of execution
2. **Executors** - frameworks for managing thread pools
3. **Locks** - mechanism for synchronized access to shared resources
4. **ForkJoinPool** - construct for recursive task splitting and merging
5. **Parallel Streams** - declarative parallel processing
6. **Virtual Threads** - optimized light-weight threads

#### Threads
In theory a thread is the smallest unit of execution that can be scheduled by OS scheduler.

In Java environment, `Thread` usually refers to the *standard* thread introduced from version 1.0 which represents the core abstraction of concurrency in Java. Internally it is a JVM-managed abstraction over an OS-level thread. The `Thread` is used to create *platform threads* that are typically mapped 1:1 to OS kernel threads. The OS allocates a large stack and other resources to platform threads (those resources are limited). 

However, using OS kernel threads for implementation fails to meet today's concurrency requirements. Two major problems:
1. Threads cannot match the scale of the domain's unit of concurrency. Today's applications allow up to millions of transactions, users or sessions. However, the number of threads supported by the kernel is much less. **Therefore, a thread for every user, transaction, or session is often not feasible**. 
2. Most concurrent applications need some synchronization between threads for every request. Due to this, **an expensive context switch happens between kernel threads**.

Any implementation of a thread depends on two constructs:
1. `Task` - a sequence of instructions that can suspend itself for some blocking operation
2. `Scheduler` - takes care of assigning the task to the CPU and reassigns the CPU from a paused task

In order to suspend a task/thread, it's required to store the entire call stack, and similarly retrieve the call stack on resuming a task/thread. This results in an expensive operation in terms of time and memory. 

Then, usually, we don't want to block standard threads and that may be solved with the use of non-blocking I/O APIs and asynchronous APIs but that may clutter the code and make debugging harder. 

For that matter [[#Virtual Thread|virtual threads]] are a better alternative in some cases.

##### Construction
There are two primary ways to create a thread:
1. Extending the `Thread` class and overriding its `run()` method
```java
class MyThread extends Thread {  

	@Override  
	public void run() {  
		System.out.println("Thread extending Thread class is running!");  
	}  
}  
  
// To start it:  
// MyThread thread = new MyThread();  
// thread.start();
```

2. Implementing the `Runnable` interface and overriding its `run()` method. Then the instance is passed to the `Thread` constructor. This is generally preferred way as it allows to extend other classes and promotes a better separation of concerns.
```java
class MyRunnable implements Runnable {  
  
    @Override  
    public void run() {  
        System.out.println("Thread implementing Runnable is running!");  
    }  
}  
  
// To start it:  
// Thread thread = new Thread(new MyRunnable());  
// thread.start();
```

To execute a thread, the `start()` method is used which creates a new thread and invokes the `run()` method that begins the code execution in that new thread. Calling `run()` directly will execute the logic, but in the **current thread**. 

##### Lifecycle
- **NEW** - created but has not yet been started (`new Thread(...)`)
- **RUNNABLE** - executing in the JVM (either currently running or ready to run, waiting for CPU window) (`thead.start()`)
- **BLOCKED** - waiting for a monitor lock to enter a synchronized block/method
- **WAITING** - waiting for another thread to perform a particular action (e.g. calling `wait()`, `join()`)
- **TIMED_WAITING** - waiting for another thread to perform an action for a specified waiting time (e.g. calling `sleep(long millis)`, `wait(long millis)`, `join(long millis)`)
- **TERMINATED** - completed its execution or has been aborted. End state.
![[Pasted image 20260121151313.png]]

#### Synchronization
One of the main concerns in concurrency is thread safety. When multiple threads access and modify shared data, we can run into a race condition. 

A **race condition** occurs when the outcome of the program on the unpredictable sequence or timing of operations performed by multiple threads. This leads with high probability to incorrect and/or inconsistent results. In short, all threads are "racing" to access/change the data.  The problem often occurs when one thread does a "check-then-act" and another thread does something to the shared value between the "check" and "act":
```java
if (x == 5) // The "Check"
{
   y = x * 2; // The "Act"

   // If another thread changed x between "if (x == 5)" and "y = x * 2" above,
   // y will not be equal to 10.
   // y could be 10, or it could be anything, you have no way of knowing.
}
```

To avoid a race condition,  the critical section must be treated with a *lock*. In java there is a built-in mechanism under `synchronized` keyword that ensures that only one thread can execute a specific block of code or method at a time. Internally it works by acquiring an intrinsic lock (also known as a *monitor lock*) associated with an object. 

When a thread calls a `synchronized` method, it acquires the lock on the `this` object. If another thread tries to call **any** `synchronized` method on the _same_ object, it will be blocked until the first thread releases the lock.
```java
// Example of synchronized method 
class Counter {  
    private int count = 0;
    
	// Locks on the 'Counter' object instance  
    public synchronized void increment() { 
        count++;  
    }  
  
    public synchronized int getCount() {  
        return count;  
    }  
}
```

Synchronized blocks are similar but allow for a more flexible control. They allow to specify which object's lock to acquire, and synchronize only the critical section of code, rather than the entire method (even though, the critical section can be extracted to a separate method).
```java
// Example of synchronized block
class AtomicCounter {  
    private int count = 0;
    // A dedicated lock object  
    private final Object lock = new Object();
  
    public void increment() { 
	    // Locks on the 'lock' object   
        synchronized (lock) { 
            count++;  
        }  
    }  
  
    public int getCount() {  
        synchronized (lock) {  
            return count;  
        }  
    }  
}
```

##### Locks
While synchronized is powerful, it uses its intrinsic lock (*monitor lock*), which is an all-or-nothing approach. It's not possible to try to acquire a such lock without blocking, nor to interrupt a thread waiting for a lock. 

**ReentrantLock** provides more flexibility and control than `synchronized`. It's the concrete implementation of the `Lock` interface. It's possible to acquire a lock without blocking, and to interrupt a thread waiting for a lock. Additionally, it can be configured to "fairness", meaning that the access is granted to the longest-waiting thread in situations that multiple threads are trying to acquire a lock.
```java
import java.util.concurrent.locks.ReentrantLock;  
  
class AdvancedCounter {  
    private int count = 0;  
    private final ReentrantLock lock = new ReentrantLock();  
  
    public void increment() {  
        lock.lock(); // Acquire the lock  
        try {  
            count++;  
        } finally {  
            lock.unlock(); // Always release the lock in a finally block!  
        }  
    }  
    // ... getCount() would also use lock/unlock  
}
```

**`volatile`** keyword doesn't provide atomicity like locks, but guarantees visibility. When a `volatile` variable is modified, that change will be visible to other threads *immediately*. Similarly, any read will always fetch the latest value from main memory, bypassing CPU caches. 
- Often useful for flags or status variables  where one thread writes and other threads read. 
- It's not suitable for compound operation (like incrementing) because two threads can read the latest value, but both would carry out the same update operation, thus the update would be effectively just one instead of two (two threads read `count = 5`, both write back `6`).

##### Communication
Threads need to coordinate their activities beyond just mutual exclusion. They might need to wait for a certain condition to be met by another thread, or signal that a condition has changed. 

**`wait()`** causes the current thread to release the lock on the object and enter the `WAITING` state until another thread invokes `notify()` or `notifyAll()` on the same object, or the specific timeout elapses. The thread then re-acquires the lock before continuing.

**`notify()`** wakes up a single thread that is waiting on this object's lock. If multiple threads are waiting, only one (arbitrarily chosen) will be notified.

**`notifyAll()`** wakes up *all* threads waiting on this object's lock.

**IMPORTANT: all methods can only be called from within a `synchronized` block or method!**

#### Virtual Threads
Light-weight threads managed by the JVM instead of the OS scheduler. They run on the *carrier thread* which is the actual kernel thread used under the hood (i.e. *platform thread*). As a result, their allocation doesn't require a system call and they're free of the expensive OS's context switch, which means it is possible to have many more virtual threads. 

Another advantage is that blocking a virtual thread doesn't block the carrier thread and that makes blocking virtual threads a much more cheaper operation - the JVM will schedule another virtual thread, leaving the carrier thread unblocked.

Ultimately, with the virtual threads the need for non-blocking I/O or async APIs disappears and that should result in more readable code that is easier to understand and debug. However, it is till possible to block the carrier thread e.g. by calling a native method and performing blocking operation from there.

Virtual threads are suitable for executing tasks that spend most of their time blocked. It is unwise to use them for long-running CPU-intensive tasks.

![[Pasted image 20260122134638.png]]<sup>Architecture overview of virtual threads</sup>

