---
date: 2021-02-11
title: "Method Chaining Base Class Methods with Derived Classes in C++"
author: Darius Grant
---
# {{ page.title }}

---

## What influenced this?
As I was refactoring my code on my personal project *[StellarVk](https://github.com/dariusgrant/StellarVK)*, I noticed one very annoying and repetive task that
I had to complete whenever adding a new class that derived from the abstract base class `UniqueObject::Builder`. That task was to implement the two exact same methods `add_flags`
and `add_next_extension` in the derived classes to allow method chaining when the implementations never change.

### Method Chaining
If you're not familiar with *[Method Chaining](https://en.wikipedia.org/wiki/Method_chaining)*, essentially it's calling multiple statements at once. I'm oversimplifying the definition but you can read on it more to get the gist of it. Below is a simple example of how one would use method chaining:

```
Bike bike = Bike()
            .setColor(red)
            .setTrainingWheels(true)
            .setBellSound(BellSound::Jingle);
```

This form of programming can decrease the amount of code written and provides readability if used correctly. Typically, method chaining is used in *[Fluent Interfaces](https://en.wikipedia.org/wiki/Fluent_interface)* and domain-specifc languages. A common example of method chaining you've already probably seen is `std::basic_ostream::operator<<`. 

## Problem Example
Without diving into the code of my project (comes later in Application section), I'm going to simplify the problem. Say that I have two classes, *Vehicle* and *Car* where *Car* derives from *Vehicle*. There's a lot
of things that all vehicles have in common, like it's make, model, year, color, etc. So naturally, these methods should be under *Vehicle*. However, what makes a car
car different from other vehicles is that they can be sedans, sports, or hatchbacks (there probably is more but for simplicity sake). 

```
class Vehicle{
public:
    Vehicle& set_make( const std::string& make_ ){
        make = make_;
        return *this;
    }
    // set model, year, and color methods
    
 protected:
    std::string make;
    // Make, Year, and Color variables
}

class Car : public Vehicle{
public:
    Car& set_type( const CarType& type_){
        type = type_;
        return *this;
    }
    
protected:
    CarType type;
}
```

Nice. We got everything implemented and expect everything to work. Let's test it out.

```
int main(){
    Car car = Car()
              .set_make( "Honda" )
              .set_type( CarType::Sedan );
              
    return 0;
}
```

What's this error? `Class "Vehicle" has no member "set_type"`? Proposterous! *Car* is inheriting class *Vehicle* so it must be able to set the make and the type, right?
It has access to the public methods to call it through inheritance so what's the problem?

The problem is the return type. Essentially, once the base class method is called, it's returning the type *Vehicle*. Unfortunately, it doesn't have any access to *Car*
and its methods. However, a workaround is to reorder the call to have *Car* methods before *Vehicle* methods but that is a very naive approach.

## Solution
To make this work in a more elegant way, we need to make use of templates. The pattern that I found when researching solutions to my problem with my project is 
***Curiously Recurring Template Pattern*** (CRTP for short). This pattern states that a class *B* derives from a template class *A* using itself as the template
argument for class *A*. Below is the general idea:

```
template <class T>
class A{
    // Functions and Variables
}

class B : public A<B>{
    // Functions and Variables
}
```

Hopefully, you can see the use of it but this is going to allow the possibility to method chain base classes with derived classes effortlessly. If we apply this
pattern to our example with vehicles, the implementation will look more closely like this:

```
template <class T>
class Vehicle{
public:
    T& set_make( const std::string& make_ ){
        make = make_;
        return static_cast<T&>(*this);
    }
    
    // ...
}

class Car : public Vehicle<Car>{
public:
    Car& set_type( ... );
}
```

What's happening now is that the methods are returning a reference to the template parameter type. So when we pass *Car* into the template, the methods are returning
a reference of type *Car*. This effectively allows chaining of base class methods mixed with derived class methods without restriction.

## Application
The example above is a very simplified demostration to show how one might be able to implement the solution. There are plenty of caveats when considering doing this, such as what happens if one try to construct `Car` by calling `Vehicle<Car>()` and chaining methods that with functions that ultimately return type `Car`? Instead of `static_cast`, maybe `dynamic_cast` would be a safer option to perform RTTI check but would require your base class to be abstract. Perhaps a protected constructor so the user can't call `Vehicle<Car>()`?

There's also the question of whether this is worth implementing or should you rethink your design. Many think that method chaining itself is bad overall for general use. But there are some cases that might be desirable, such as implementing the Builder pattern which was my use case. Below is an example program, similiar to my project, of how I implemented the solution after making some careful considerations based on some constructive criticism. 

```
#include <memory>
#include <string>
#include <iostream>

// Arbitrary object that has some common member variables for derived classes.
class Object{
public:
    // Abstract class - prevents user from instantiating this
    template<class Self, class ClassToBuild>
    class ObjectBuilder{
    protected:
        std::unique_ptr<ClassToBuild> classToBuild;
        
    public:
        Self& set_a( const int a_ ) {
            classToBuild->a = a_;
            return dynamic_cast<Self&>( *this );
        }
        
        Self& set_b( const int b_ ) {
            classToBuild->b = b_;
            return dynamic_cast<Self&>( *this );
        }
        
        virtual ClassToBuild&& build() = 0;
    };
    
protected:
    // Common object members.
    int a;
    int b;
    // ...

public:
    // Getters for variables
    
protected:
    // Protected to prevent user from instantiating directly and
    // only by the builder.
    Object() = default;
};

class ConcreteObject : public Object{
public:
    class ConcreteObjectBuilder : public ObjectBuilder<ConcreteObjectBuilder, ConcreteObject> {
    public:
        ConcreteObjectBuilder( const bool requiredBool_ ) {
            classToBuild = std::unique_ptr<ConcreteObject>( new ConcreteObject( requiredBool_ ) );
        }
        
        ConcreteObjectBuilder& set_str( const std::string& str_ ) {
            classToBuild->str = str_;
            return *this;
        }
        
        ConcreteObject&& build() override {
            return std::move( *classToBuild );
        }
    };
    
protected:
    // boolVar required for construction
    const bool boolVar; 
    std::string str;
    
public:
    // Getters for variables
    
    void print(){
        std::cout << "a: " << a << ", b: " << b << ", boolVar: " << boolVar << ", str: " << str  << std::endl;
    }

protected:
    ConcreteObject( const bool requiredBool_ ) : boolVar( requiredBool_ ){}     
};

int main(){
    ConcreteObject obj = ConcreteObject::ConcreteObjectBuilder( true ).set_a( 1 ).set_str( "Hello" ).build();
    obj.print();
}
```

There's no way the object can be created directly without using the builder. The builder ensures that it is built in its entirety and correctly by enforcing a constructor that matches the concrete object's constructor. Logically, any variable that is required to construct a valid object should be included in the constructor. Otherwise, it's optional to call other member functions to alter the value of variables as the default is also valid to construct a valid object. Finally, calling `build()` transfers ownership from the builder to the variable you're giving it to. Take a look for yourself in [godbolt](https://godbolt.org/z/shEnx3).

## Benefits and Limitations
As mentioned already, the benefit is to mix method chaining with base and derived classes. It reduces the amount of repetive code that would've been needed to achieve
the same thing without templates by definining a method once in the base class. Before learning about this, I marked the base methods as virtual and implemented the
same piece of code in derived classes to achieve method chaining. Obviously, this violates the DRY principle. 

A primary limitation most will face is that there's not much use for implementing this without good cause. There aren't very many use cases for why one might need to use method chaining along with CRTP, at least plausible cases. The builder pattern makes sense of why method chaining would be useful. But for most developers, I'd assume that this would find any merit outside of very specific situations. Some deem method chaining as nothing more than syntatic sugar and a minor conveninece that causes more issues than solving them.

In regards to using the pattern itself, a specific limitation to consider is that multiple inheritance levels is very tricky to get right from what I've discovered. From my testing, one can probably achieve method chaining by having a template argument for the class itself and an argument for the derived class. The outline would be something like below:

```
template <class Self, class Derived>
class Vehicle{
    public:
        Derived& a(){}
}

template <class Self, class Derived>
class Car : public Vehicle<Car<Self, Derived>, Derived>{
    public:
        Derived& b(){ ... }
}

template <class Self, class Derived>
class RaceCar : public Car<RaceCar<Self, Derived>, Derived>{
    public:
        Derived& c() { ... }
}

class F1RaceCar : public RaceCar<F1RaceCar, F1RaceCar>{
    public:
        F1RaceCar& d()) { ... }
}
```

## Thank you for reading!
Thank you for reading my post! These posts are inspired by actual problems I face when developing so it's fun to look over the things I've learned and put it out for
anyone that may find it useful. I also edit my posts after consideration of constructive critism to make sure I do not misguide readers. The last thing I want is your understanding to be naive and your code crashing because of my content. I'd like to set higher bars for myself than to allow that to happen. Any and all constructive critism is welcomed!

Please consider supporting me by adding me on LinkedIn, Github, and/or donating as I continue to progress as a developer.

[LinkedIn](https://www.linkedin.com/in/darius-grant-783614190)

[Github](https://github.com/dariusgrant)

[Paypal](https://www.paypal.com/paypalme/DGrant230)

Venmo: @DGrant230

CashApp: $DGrant230
