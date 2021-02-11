---
date: 2021-02-11
title: "Method Chaining Base Class Methods with Derived Classes in C++"
author: Darius Grant
---
# {{ page.title }}

---

## What influenced this?
As I was refactoring my code on my personal project *[StellarVk](https://github.com/dariusgrant/StellarVK)*, I noticed one very annoying and repetive task that
I had to complete whenever adding a new class that derived from the base class `UniqueObject::Builder`. That task was to implement the two exact same methods `add_flags`
and `add_next_extension` in the derived classes to allow method chaining when the implementations never change.

### Method Chaining
If you're not familiar with method chaining, essentially it's calling multiple statements at once. I'm oversimplifying the definition and you can read on it more by
a google search but below is a simple example of how would use method chaining:

```
Bike bike = Bike()
            .setColor(red)
            .setTrainingWheels(true)
            .setBellSound(BellSound::Jingle);
```
This form of programming can decrease the amount of code written and provides readability if used correctly.  I won't be going over the benefits and drawbacks as this
isn't the focus of this post. 

## Problem Example
Without diving into the code of my project, I'm going to simply the problem. Say that I have two classes, *Vehicle* and *Car* where *Car* derives from *Vehicle*. There's a lot
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

## Benefits and Limitations
As mentioned already, the benefit is to mix method chaining with base and derived classes. It reduces the amount of repetive code that would've been needed to achieve
the same thing without templates by definining a method once in the base class. Before learning about this, I marked the base methods as virtual and implemented the
same piece of code in derived classes to achieve method chaining. 

A specific limitation to consider is that multiple inheritance levels is very tricky to get right from what I've discovered. From my testing, one can probably achieve
method chaining by having a template argument for the class itself and an argument for the derived class. The outline would be something like below:

```
template <class Derived>
class Vehicle{
    public:
        Derived& a(){}
}

template <class Self, class Derived>
class Car : public Vehicle<Car<Self, Derived>>{
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

It's can get pretty confusing but it's possible. As of now, I can't think of other possible limitations so if you do, please let me know so I can add it to my arsenal
of knowledge.

## Thank you for reading!
Thank you for reading my post! These posts are inspired by actual problems I face when developing so it's fun to look over the things I've learned and put it out for
anyone that may find it useful. Please consider supporting me by adding me on LinkedIn, Github, and/or donating as I continue to progress as a developer.

[LinkedIn](https://www.linkedin.com/in/darius-grant-783614190)

[Github](https://github.com/dariusgrant)

[Paypal](https://www.paypal.com/paypalme/DGrant230)

Venmo: @DGrant230

CashApp: $DGrant230
