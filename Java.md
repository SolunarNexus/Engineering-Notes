> *Mainly notes about internal details of how Java does the magic.*
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