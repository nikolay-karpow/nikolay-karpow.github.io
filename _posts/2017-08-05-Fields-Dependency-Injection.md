---
layout: post
title:  "Fields dependency injection"
date:   2017-08-05 10:15:00 -0600
categories: spring java
comments: true
---
There are couple of different types of dependency injection in the Spring framework. Not all of them are equally good. And, unfortunately, it looks like the worst of them is the most popular.

Let's start by highlighting some important qualities of the code, which different DI types makes more difficult or easier to achieve.
#### 1. Testability

   Code must be easy to test by unit tests. DI must not prevent working in TDD way. It means, objects have to support creation without any DI container at all, otherwise tests are slow.
#### 2. Immutability

   Immutable objects are easier to deal with, than mutable ones. They are less error-prone, thread safe, easier to reason about. It means, DI shouldn't prevent the fields to be final.
#### 3. Reusability

   Code should be easy to reuse in any environment, no matter what's used for application assembly (one DI framework or another, or even no DI container at all). It again means, that objects have to support creation without DI container.

## Constructor injection
Let's start with the best one. All the dependency are passed as constructor parameters.
{% highlight java %}
class ConstructorInjectionExample {

    private final Bean bean;

    @Autowired
    public ConstructorInjectionExample(Bean bean) {
        this.bean = bean;
    }
}
{% endhighlight %}
In my opinion, it has only advantages.
* It doesn't disrupt testing, as you can create an instance of the class in a plain java way
* It allows tests to be fast as you don't have to wait until Spring fires up
* It allows to use final modifier
* It doesn't depend on Spring, you can reuse this class anywhere you want

One might argue, that it's too verbose. It's verbose, but it's explicit and it has all the advantages mentioned above.

## Setter injection
This one is a bit worse. Dependencies are set via setter methods.
{% highlight java %}
class SetterInjectionExample {

    private Bean bean;

    @Autowired
    public void setBean(Bean bean) {
        this.bean = bean;
    }
}
{% endhighlight %}
It has the same characteristics as the constructor injection, except for the immutability support.
Disadvantages:
* Doesn't support immutability
* It makes the dependencies less explicit.
  When you use constructor injection and you wanna create an instance of the class, all you have to specify is in the arguments list, everything is explicit. When you use setters and you need an instance, you have to look at the methods: which setters do you need to call before you can use the instance?

## Fields injection
This is the worst one.
{% highlight java %}
class FieldsInjectionExample {

    @Autowired
    private Bean bean;

}
{% endhighlight %}
The only advantage:
* It's concise

Disadvantages:
* It doesn't allow to create an object without DI container
* It makes testing difficult, especially for the TDD approach as the feedback loop is long because of slow tests
* It makes reuse difficult as there is a dependency on the DI container
* It doesn't support immutability

## Conclusion
I strongly advice you to use constructor injection whenever possible. I don't see any justification for fields injection, don't use it. Ever! In fact, even IntelliJ IDEA's inspection suggests avoiding it:

![]({{site.github.url}}/assets/2017-08-06-Idea-advice-against-fields-injection.jpg)
