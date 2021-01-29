---
date: 2021-01-29
title: "Store, Retrieve, and Remove an Arbitrary Type in a Map in C++"
author: Darius Grant
---
# {{ page.title }}

***Disclaimer***: This is my first blog post! I'm excited to share with whatever I come across as I continue to develop my skills but this is also by no means professional.
So don't expect what you may usually find in some others. Though, I will strive to become better at blogging and bring a better experience to the readers.

---

## What influenced this?
During the time I've been working on my personal project developing with Vulkan C++ headers, including extensions became more convenient with the use of `vk::StructureChain<T...>`. For those who aren't familiar with Vulkan, it's an API that allows low-level access to the GPU to perform 3D graphics and/or computing. The underlying structure of `vk::StructureChain<T...>` is an `std::tuple<T...>` as that is what it inherits from.

However, an issue I ran into is that I wanted to dynamically add to the chain. Maybe there's a check for a certain set of extensions that I wanted to include that was available to the GPU. But I needed to know all the types I wanted to add by compile-time if I was to add them to the tuple. Maybe if C++ had a better reflection system, this could've been made possible.

## What is it that can make this possible?
After decades (at least it felt that long) of researching and failing at producing a solution, I stumbled across two key candidates that could make this work: `std::any` and `std::type_info::hash_code`. I played around with these options in various ways, such as trying to retrieve a type based on the hash code to cast the `std::any` object and trying to force a solution into a tuple. However, the issue that persisted is that there was no way to retrieve a type from a the hash code so that I could cast the `std::any` object to it's respective type.

A little more research and fiddling came as a positive result though. Due to my use-case with Vulkan, I didn't need more than one of the same type to be in the structure. This made it much simpler as now, all I need is a way to identify them and I can store them in `std::map` and have their hash code as the key.

```
#include <map>
#include <any>

class ArbitraryMap {
  private:
    std::map<std::size_t, std::any> m;
    
  public:
    // Implementations for add, get, and remove
}
```

Okay, so the idea is there but how do we actually implement this? Well it's pretty simple actually. All we need to take into account is using function templates so that it accepts any type that you throw into it. After that, it's pretty straightforward implementing adding, getting, and removing the objects stored.

```
template <class T>
void add( T t_){
  auto hash = typeid(T).hash_code();
  m[hash] = t_;
}

template <class T>
T get( T t_){
  auto obj = map.at( typeid(T).hash_code() );
  return std::any_cast<T>( obj );
}

template <class T>
void remove( T t_){
  map.erase( typeid(T).hash_code() );
}
```

Of course there's some checks that must happen to make sure you're not casting an object that doesn't exist but that's trivial and should be handled accordingly. I personally chose to handle getting an object by using `std::optional` but it's also because I'm trying to use what I find while journeying through the standard library. The right tools are there.

## What are the benefits and limitations?
Obviously, the biggest benefit is that you can use this with any type dynamically without knowing what you need to add by compile-time. This is huge since C++ lack of reflection holds back a lot of potential for great development. Another benefit is that it's stored in an ordered map as the hash codes are the keys, which makes access logarithmic. It's also extendable to have more than one of the same type by using another data structure as the value such as `std::vector`. 

The limitations that this is that iterating isn't possible. You need to know what you want from the map before retrieving it. Unless there's a way to convert a type's hash code into its type, then you're stuck with manually keeping track of what you have in there. So technically, it is possible to iterate but very tedious.

## Thank you for reading my first post ever!
I really appreciate you for making this far! I'd like to continue making this type of blogs as I continue with progressing as a developer and professional (whenever I get an actual job in developing anyway). It's a way of investing in myself as I document my times with tough problems and unique solutions and whatever else I think may be of interest to you all. Please consider donating if you'd like to support me. 

[Paypal](https://www.paypal.com/paypalme/DGrant230)

Venmo: @DGrant230

CashApp: $DGrant230
