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

### Data Encapsulation
### Inheritance