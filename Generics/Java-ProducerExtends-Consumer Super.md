[Java: Producer Extends, Consumer Super](https://medium.com/@isuru89/java-producer-extends-consumer-super-9fbb0e7dd268)
======================================

[![Isuru Weerarathna](https://miro.medium.com/v2/resize:fill:88:88/1*9j_2GbFAWKrzu4ouGgb0Gw.jpeg)](https://medium.com/@isuru89?source=post_page-----9fbb0e7dd268--------------------------------)

[Isuru Weerarathna](https://medium.com/@isuru89?source=post_page-----9fbb0e7dd268--------------------------------)

[](https://medium.com/plans?dimension=post_audio_button&postId=9fbb0e7dd268&source=upgrade_membership---post_audio_button----------------------------------)

Java is a statically typed language, and then there are generics.

![](https://miro.medium.com/v2/resize:fit:700/0*hIrvJFUOGnirF4N5)

Photo by [Pritesh Sudra](https://unsplash.com/@pritesh557?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

Recently I was experimenting with [Flink](https://flink.apache.org/), which is a stateful stream processing framework, to create a small stream pipeline for one of my applications. When defining streams and its operators, it uses static typing extensively to prevent runtime failures, hence leading to reliable execution handling. Not so surprisingly, its programming model is heavily built on top of generics.

Java generics is all about providing type safety. It allows creating reusable types composing other types as parameters. Or, in coding terminology, it prevents accidental `ClassCastException` when dealing with type parameters.

For example, think about generic `List<E>` class. Usually, Java does not know what we are going to store in that list and that list must be reusable too. So, it allows us to specify the list parameter type when an instance is being created. Therefore Java, at compile-time, validates whether we are actually using the list only for that type and if it violates, a compilation error will be raised. In the below line, it allows only to store `Vehicle` or any sub-types of `Vehicle` inside the list.

```java
List<Vehicle> myVehicles = new ArrayList<>();
```

> Introduction with Java 7 diamond syntax, we can ignore repeating type information in the right hand side of the assignment.

Wildcard Generics
-----------------

Using wildcard `?`, a programmer can restrict the composed parameter type(s) that allows in the parent type. For example, if you have a class called `MyList` which accepts only certain subtypes of a class, say `Vehicle`, then it can be declared as below.
```java
public static class MyList<E extends Vehicle> implements List<E> {
    // ...
}
```
Here, users allow adding only the subtypes of the `Vehicle` classes to the `MyList`. Java will prevent anything other than a `Vehicle` type being added in anywhere of the program.

I have shown you how a generic is being used in a class definition. This is getting interesting when the generics are going to use in method definitions. More specifically, in arguments.

In some situations, especially when we are designing or using libraries or frameworks, we had to deal with passing such generic types to methods. Passing a subset or a superset of types using generics takes us in a very interesting direction. Here we are going to meet, [*Covariance*](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)) (`? extends E`) and [*Contravariance*](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)) (`? super E`). And this is where the title of the article comes (*Producer Extends, Consumer Super*), or PECS.

Invariance <E>
--------------

Before diving into the other modes, first let's deal with the known one, *Invariance*. You have seen this mode always. It is just the type parameter without any wildcards.
```java
List<Vehicle> garage = new ArrayList<>();
garage.add(new Car());
garage.add(new Bus());

// reading behavior
Vehicle firstVehicle = garage.get(0);
```
In here, you can insert any subtypes of Vehicle class into the garage list. And when you reading it, you will return a type of Vehicle instance.

But this is called Invariance because you can't do the below assignment.
```java
List<Car> cars = new ArrayList<>();
List<Vehicle> garage = cars;        // compilation error
```
Because, in invariance, "list of cars" is not a "list of vehicles", and "list of vehicles" is not a "list of cars", though `Car` is a `Vehicle`. You cannot assign each other because the parameter type is *invariant* on a specified `List<E>` type.

This is known in generics. And hence, no surprises.

Upper Bound Wildcard <? extends E>
----------------------------------

Now declare the garage variable as `List<? extends Vehicle>`. It will still return an instance of `Vehicle` when reading, but surprisingly you cannot insert items to the list, not even a subtype of `Vehicle`.
```java
List<? extends Vehicle> garage = new ArrayList<>();
garage.add(new Vehicle());  // compilation error
garage.add(new Car());      // compilation error
garage.add(new Bus());      // compilation error

// reading behavior
Vehicle vehicle = garageB.get(1);
```
What's happening here? Why?

As soon as you declare a list parameter type with `extends` then it becomes a producer to other variables. That means with the extends keyword, Java compiler knows that this list could contain any subtype of `Vehicle` class and it does not know which kind due to there can be many subtypes of it. Hence compiler cannot erase type to a definite one. So, to preserve the type-safety, user does not allow to insert any kind of items to it.

But, when you get an item from it, the list knows and guarantees that every item in the list is a sub-type of `Vehicle`. Because it is written as any type extends from `Vehicle`. So it can surely *produce* a type of `Vehicle` out of the list.

So, from the list perspective, it acts as a producer to others. You can get items from it (list produces), but you can't insert into it. In Java world, it is called *covariance*.

> Covariance exhibits the producer behavior from that type's perspective

Then what's the use of it, seriously?

You usually should not be declaring variables like above when you are coding. The reason is that this producer's behavior is useful only when they are being passed as method arguments. If a method only allows reading items from a list, then you can declare the list parameter using extends.

What if I want to add any sub-types of `Vehicle` to the list?

Lower Bound Wildcard <? super Vehicle>
--------------------------------------

Here comes the consumer mode. This is also called *contravariance*. Let me explain this by a similar example to the above.
```java
List<? super Car> garage = new ArrayList<>();
garage.add(new BMW());
garage.add(new Alto());
garage.add(new Vehicle());    // compilation error

// reading behavior
Object object = garage.get(0);    // I don't get a Car, why?
```
If you declare the type parameter using super keyword, then Java compiler knows that the `garage` variable contains references to any supertype of `Car` class. But again, it does not know which supertype is. Remember, any subtype can be inserted into a collection defined with supertype. Therefore, you can add any subtype of `Car` into this list, but not superclasses (`Vehicle`).

Since the list does not know which kind of supertypes it could contain, Java will return only an `Object` type to assure type safety when you read from the list, which is not useful at all. Returns `Object` type because it is the root of every class. If you are reading from it, you have to know the type of the instance when accessing it. If you don't know it, you can't consume it.

This is called consumer behavior, because, from the list perspective, it allows to add items to it (list consumes), but not useful in type-safety reading (producing).

> Contravariance exhibits consumer behavior from that type's perspective

Similar to the producer, this signature is being used in method arguments where the method supposed to insert items into it.

Assignability
-------------

Now you might ask what if a method wants to both produce and consume from a list?

Easy, don't use wildcards! Just use *invariance* mode of the parameter, i.e. `List<Vehicle>`. It implicitly acts as both producer and consumer with type safety.
```java
public static void main(String[] args) {
    List<Vehicle> garage = new ArrayList<>();
    Car car = new Car(); 
    accept(garage, vehicle);
    repair(garage);
}

public static void repair(List<? extends Vehicle> list) {
    // repair each vehicle one by one
}

public static void accept(List<? super Vehicle> list, Vehicle v) {
    // register a vehicle for repairing in garage
}
```
As you can see from the above example, the following is assignable.
```java
List<Vehicle> garage = // ...
List<? extends Vehicle> garageB = garage;
```
And the following too.
```java
List<Vehicle> garage = // ...
List<? super Vehicle> garageC = garage;
```
That means, once you have an invariance mode of variable, then parameter mode does not matter.

Why there are two wildcard modes?

The reason is type safety. If you are a one who develops frameworks or libraries which are going to be used by others, then you can use *Covariance* and *Contravariance* to provide the type-safety to the framework users. Method generic argument(s) can be defined in a way to indicate to a user what kind of operations are going to be done within the method using those generic arguments.

Do I need to use this when I defining methods?

No. If you did not need to use these modes so far, then you won't need too. The only usage I see is that it happens when you actually going to use a library written in this way. Then you need to understand what kind of arguments expecting from these library methods and provide them correctly.

Unbounded Wildcards <?>
-----------------------

Unbounded wildcards do not use a type in its parameter at all.
```java
List<?> garage = new ArrayList<>();
garage.add(1);               // compilation error
garage.add(new Object());    // compilation error
garage.add(new Car());       // compilation error

// reading behavior
Object o = list.get(0);
```
Having wildcard only, I can't add any item of any type of it. When I read an item, it gives me an `Object` reference, which is not useful.

This type of wildcard is useful when your method actually does not care about its items at all. For example, think you have a utility method that checks whether a given Collection is empty or not. In there we don't worry what the collection contains, but we care only about properties of the wrapping reference.
```java
public static boolean isNullOrEmpty(Collection<?> coll) {
    return coll == null || coll.isEmpty();
}
```
Unless it is very specific situations like the above method, you should always try to use the true potential of Java generics as much as possible since generics are natural protectors of type misuses.

Java generics are really useful when dealing with data structures and data operators, and it becomes a mandatory skill when you are continuing your career through Java or any other type-checking language. Wildcards are a method to relax type constraints when using generics and extensively used in libraries and frameworks. It is easy to remember which wildcard to use at which occasion using the word acronym PECS, Producer Extends, Consumer Super.