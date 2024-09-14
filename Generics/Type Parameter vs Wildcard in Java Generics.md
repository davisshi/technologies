[Type Parameter vs Wildcard in Java Generics](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard)
===========================================

1\. Overview[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#overview)
------------------------------------------------------------------------------------------

In this tutorial, we'll discuss the differences between formal type parameters and wildcards in Java [generics](https://www.baeldung.com/java-generics) and how to use them properly.

It's a common mistake to think they are the same. When writing a generic method, we often think whether we should use a type parameter or a wildcard. In this tutorial, we'll try to find the answer to this question.

2\. Generic Classes[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#generic-classes)
--------------------------------------------------------------------------------------------------------

In many programming languages, a type can be parameterized by another type. This is a feature known as generics in Java or Parametric Polymorphism in other languages.

Generics were introduced in Java 5. Using them, we can create classes and methods that differ only in type. Furthermore, we can create reusable code.

When introducing generics in our classes and interfaces we should use type parameters.

As an example, let's see the [*java.lang.Comparable*](https://www.baeldung.com/java-comparator-comparable#comparable) interface:

```
public interface Comparable<T> {
    public int compareTo(T o);
}
```

The type parameter defines the formal type, usually named with one uppercase letter (e.g. *T*, *E*). Here, the parameter *T* is available throughout the *Comparable* interface.

When instantiating, we need to provide a concrete type (type argument). We can't use wildcards to define a generic class or interface.

3\. Generic Methods[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#generic-methods)
--------------------------------------------------------------------------------------------------------

The most frequent way to use generics is for common static methods since they aren't part of an instance that can declare the type. We often use them to create libraries or to provide an API to our clients.

### 3.1. Method Parameters[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#1-method-parameters)

To write a method with a generic type argument, we should use a type parameter. So let's create a method that prints a given *item*:

```
public static <T> void print(T item){
    System.out.println(item);
}
```

In this example, we aren't able to replace the type argument *T* with the wildcard. We can't use wildcards directly to specify the type of a parameter in a method. The only place we can use them is as part of a generic code, for example, as a generic type argument defined in the angle brackets.

There's often a case when we can declare a generic method using either wildcards or type parameters. For instance, here are two possible declarations of a *swap()* method:

```
public static <E> void swap(List<E> list, int src, int des);
public static void swap(List<?> list, int src, int des);
```

The first method uses an unbounded type parameter while the second one uses an unbounded wildcard. Whenever we have an unbounded generic type, we should prefer the second declaration.

Wildcards make code simpler and more flexible. We can pass any list and, additionally, we don't have to worry about type parameters. If a type parameter appears only once in the method declaration, we should consider replacing it with a wildcard. The same rule applies to bounded type parameters as well.

Without upper or lower bounds, the wildcard represents "any type", or a type of unknown. Its purpose is to allow a variety of actual argument types to be used at different method invocations. Moreover, wildcards are designed to support flexible subtyping.

### 3.2. Return Types[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#2-return-types)

Now, let's look at a *merge()* method that returns a wildcard type:

```
public static <E> List<? extends E> mergeWildcard(List<? extends E> listOne, List<? extends E> listTwo) {
    return Stream.concat(listOne.stream(), listTwo.stream())
            .collect(Collectors.toList());
}
```

Say we have two lists of *Number*s we'd like to merge:

```
List<Number> numbers1 = new ArrayList<>();
numbers1.add(5);
numbers1.add(10L);

List<Number> numbers2 = new ArrayList<>();
numbers2.add(15f);
numbers2.add(20.0);
```

Since we are passing two lists of *Number*s, we'd expect to receive the same type of *List* back. This wouldn't be the case if we have wildcard type as the return type. The code below wouldn't compile:

```
List<Number> numbersMerged = CollectionUtils.mergeWildcard(numbers1, numbers2);
```

Instead of providing additional flexibility, using wildcards would force clients to deal with them by themselves. When used properly, the client shouldn't be aware of wildcard usage. Otherwise, it may indicate we have a design problem in our code.

When a generic method returns a generic type, we should use a type parameter instead of a wildcard:

```
public static <E> List<E> mergeTypeParameter(List<? extends E> listOne, List<? extends E> listTwo) {
    return Stream.concat(listOne.stream(), listTwo.stream())
            .collect(Collectors.toList());
}
```

With the new implementation, we can store the result in the list of *Number* elements.

4\. Bounds[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#bounds)
--------------------------------------------------------------------------------------

Generic-type bounding allows us to restrict what types can be used instead of the generic type. This Java feature makes it possible to treat generics polymorphically.

For instance, a method that operates on numbers might only want to accept instances of the *Number* class or its subclasses. This is what bounded type parameters are for.

We can use wildcards with bounds in three ways:

-   Unbounded Wildcards: *List<?>* -- represents a list of any type
-   Upper Bounded Wildcards: *List<? extends Number>* -- represents a list of *Number* or its subtypes (for instance, *Double* or *Integer*).
-   Lower Bounded Wildcards: *List<? super Integer>* -- represents a list of *Integer* or its supertypes, *Number,* and *Object*

On the other hand, a bounded parameter type is a generic type that specifies a bound for a generic. We can bound type parameters in two ways:

-   Unbounded Type Parameter: *List<T>* represents a list of type *T*
-   Bounded Type Parameter: *List<T extends Number & Comparable>* represents a list of *Number* or its subtypes such as *Integer* and *Double* that implement the *Comparable* interface

We can't use type parameters with the lower bound. Furthermore, type parameters can have multiple bounds, while wildcards can't.

### 4.1. Upper-Bounded Types[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#1-upper-bounded-types)

In generics, parameterized types are invariant. In other words, we know that, for instance, the *Long* class is a subtype of a *Number* class. However, it's important to understand that *List<Long>* isn't a subtype of *List<Number>*. To better understand the latter, let's create a method that sums up the values of elements inside the collection.

Say we want to sum up any subtype of a *Number* class. Without generics, our implementation might look like this:

```
public static long sum(List<Number> numbers) {
    return numbers.stream().mapToLong(Number::longValue).sum();
}
```

Now, let's create a list of *Number* elements and call the method:

```
List<Number> numbers = new ArrayList<>();
numbers.add(5);
numbers.add(10L);
numbers.add(15f);
numbers.add(20.0);
CollectionUtils.sum(numbers);
```

Here, everything works as expected. However, if we want to pass a list that only contains *Integer* elements, we get a compiler error. Even though an *Integer* is a subtype of a *Number*, *List<Integer>* is not a subtype of *List<Number>*.

To clarify, we aren't able to pass a list of *Integer* types because we'd be mapping two incompatible types. The *List<Integer>* and *List<Number>* aren't related like *Integer* and *Number* are, they only share the common parent (*List<?>*).

To solve this problem, we can use the wildcard with an upper bound. This type of bound uses the keyword *extends* and it specifies that the generic type must be a subtype of a given class of the class itself.

Firstly, let's modify the method using wildcards:

```
public static long sumWildcard(List<? extends Number> numbers) {
    return numbers.stream().mapToLong(Number::longValue).sum();
}
```

Next, we can call the method with a list that contains subtypes of the *Number* class:

```
List<Integer> integers = new ArrayList<>();
integers.add(5);
integers.add(10);
CollectionUtils.sumWildcard(integers);
```

In this example, the *List<Integer>* is a type of the *List<? extends Number>* which makes the call of the method valid and type-safe.

Again, we could accomplish the same functionality using type parameters:

```
public static <T extends Number> long sumTypeParameter(List<T> numbers) {
    return numbers.stream().mapToLong(Number::longValue).sum();
}

```

Here, the type of elements in a list becomes a type argument of the method. It has the name *T*, but it remains unspecified and limited with an upper bound (*<T extends Number>*).

### 4.2. Lower-Bounded Types[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#2-lower-bounded-types)

This type of bound can be used only with a wildcard since type parameters don't support them. It is expressed using the *super* keyword and it specifies the lower class in the hierarchy that can be used as a generic type.

Suppose we want to write a generic method that adds numbers to a list. Say we don't want to support decimal numbers but only *Integer*. To maximize flexibility, we want to allow users to call our method using the list of *Integer* and all of its supertypes (*Number* or *Object*). In other words, anything that can hold *Integer* values. We should use a lower-bounded wildcard:

```
public static void addNumber(List<? super Integer> list, Integer number) {
    list.add(number);
}
```

Here, any subtype can be inserted into a collection defined with a supertype.

If we are unsure whether we should use the upper or lower bound, we can think about the [PECS](https://www.baeldung.com/java-generics-interview-questions#q13-when-would-you-choose-to-use-a-lower-bounded-type-vs-an-upper-bounded-type) -- Producer Extends, Consumer Super.

Whenever our method consumes a collection, for example adding elements, we should use a lower bound. On the other hand, if our method only reads elements, we should use the upper bound. Additionally, if our method does both, i.e., it produces and consumes a collection, we can't apply the PECS rule and should use unbounded types instead.

To clarify, let's apply the PECS rule to our example. Our generic method modifies the list by adding elements to it. From the list perspective, it allows adding elements but it isn't safe to read elements. We would get a compiler error if we try to replace the lower bound with an upper bound.

### 4.3. Unbounded Types[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#3-unbounded-types)

Sometimes, we'd like to create a generic method that modifies the collection.

We already mentioned we should consider using wildcards instead of type parameters to increase flexibility. However, using wildcards for a method that supports collection modification can be tricky.

Let's consider the implementation of the *swap()* method mentioned earlier. The straightforward implementation of the method won't compile:

```
public static void swap(List<?> list, int srcIndex, int destIndex) {
    list.set(srcIndex, list.set(destIndex, list.get(srcIndex)));
}
```

The code won't compile because we declared the list to be of any type, so Java wants to save us from ourselves by forbidding any modification of a list that can contain any element. When the *set()* method is called, the compiler isn't able to determine the type of object being inserted into the list. Therefore it produces an error. This way, Java enforces type safety at compile time.

Additionally, we cannot use bounds since our method produces (reads) and consumes (updates) the list. In other words, the PECS rule cannot be applied here and we should use an unbounded type.

We could solve the problem by writing a generic helper method to capture the wildcard type:

```
private static <E> void swapHelper(List<E> list, int src, int des) {
    list.set(src, list.set(des, list.get(src)));
}
```

The *swapHelper()* method knows that list is a *List<E>*. Therefore, it knows any value it gets out of this list is of type *E* and it's safe to put any value of type *E* back into the list.

### 4.4. Multiple Bounds[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#4-multiple-bounds)

A type parameter is useful when we want to restrict the type with multiple bounds.

Firstly, let's create a simple hierarchy. Let's define an *Animal* class:

```
abstract class Animal {

    protected final String type;
    protected final String name;

    protected Animal(String type, String name) {
        this.type = type;
        this.name = name;
    }

    abstract String makeSound();
}

```

Secondly, let's create two concrete classes:

```
class Dog extends Animal {

    public Dog(String type, String name) {
        super(type, name);
    }

    @Override
    public String makeSound() {
        return "Wuf";
    }

}
```

Furthermore, the second concrete class implements a *Comparable* interface as well:

```
class Cat extends Animal implements Comparable<Cat> {
    public Cat(String type, String name) {
        super(type, name);
    }

    @Override
    public String makeSound() {
        return "Meow";
    }

    @Override
    public int compareTo(@NotNull Cat cat) {
        return this.getName().length() - cat.getName().length();
    }
}
```

Finally, let's define a method that sorts elements in the given list and requires values to be comparable:

```
public static <T extends Animal & Comparable<T>> void order(List<T> list) {
    list.sort(Comparable::compareTo);
}
```

This way, our list cannot contain elements of the *Dog* type since the class doesn't implement a *Comparable* interface.

5\. Conclusion[](https://www.baeldung.com/java-generics-type-parameter-vs-wildcard#conclusion)
----------------------------------------------------------------------------------------------

In this article, we discussed the differences between type parameters and wildcards in Java generics. To sum up, we should prefer wildcards over type parameters when writing a library for wide use. We should remember the basic PECS rule when deciding what bound type to use.

As always, the source code for the examples is available [over on GitHub](https://github.com/eugenp/tutorials/tree/master/core-java-modules/core-java-lang-5).