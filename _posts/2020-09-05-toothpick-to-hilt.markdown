---
layout: posts
title:  "Migrating from Toothpick to Dagger Hilt"
date:   2020-09-05 15:00:00 -0400
categories: dagger hilt kotlin java android
---

A little over a year ago, I started a new position as a Senior Mobile Developer and inherited a codebase that had some issues (as is typical with large software codebases). Over the year I have started many initiatives to modernize our codebase and resolve these issues (with varying levels of value at the end): migration from Java to Kotlin, migration from RxJava1 to RxJava 3 and Kotlin Co-Routines, the use of LiveData, etc. Each of these experiences have helped me learn and grow as an Android developer, and I will write articles on each of these separately.

Today, I am going to talk about Toothpick (a dependency injection library) and Hilt. When I moved to this position, I had never before used dependency injection in a project as most had been small simple state machines written in Java. It took a few weeks to get truly comfortable with the dependency injection library we used and understand how it works as the documentation for Toothpick wasn’t nearly as extensive as a library like Dagger. The toothpick library tries to abstract and make an easier version of dagger, at the cost of some complexity and runtime performance issues.

## Dependency Injection
Imagine for a moment you are going to build a sandwich; how do you get to the final product? You can’t build a sandwich without the ingredients as the sandwich depends on these (making the ingredients dependencies of a sandwitch). Now imagine this sandwich we want to build has Bacon, Lettuce and Tomato. When we construct the sandwich, we are going to provide bacon, lettuce and tomato to the sandwich; we don’t need or want to know how to actually make bacon, lettuce or tomato at this point - we just want to provide those to the sandwich. Now consider this scenario, your culinary skills have improved and now you want to build another sandwich, a CLUB. The club is chicken and lettuce under bacon. We already know lettuce and bacon so we can share the same lettuce and bacon from the BLT sandwich as they do not change. Now to build a new sandwich we only need to figure out how to make / get chicken! This process is just like dependency injection; experiences can be built with a well-known set of components much more quickly as requirements change. You can apply these tactics manually without any library or tooling but there is an enormous overhead to keeping your components up to date as more functionality is added.

## Toothpick
Toothpick strives for simplicity, which can also be a downfall. Scoping is supported, but it isn’t tied to the Android lifecycle unless you write this functionality yourself. There are two ways to create a dependency for use in your application: creating an interface and implementation or creating an implementation without any interface contract.

Interfacing has value in complex scenarios as well as in testing. When you create an interface, you must are defining a contract that is enforced by the compiler. Each scope can have its own implementation if that is a requirement: example, you have an AppStateProvider which has two methods

{% highlight kotlin %}
interface AppStateProvider {
    val currentState: EnumOfState
    fun moveToNextState()
} 
{% endhighlight %}

Now, to use this contract you must create an implementation. You can actually create many implementations, one for each sate (logged out, authenticated, background, etc). Now you will bind each implementation to the desired scope in toothpick via a module. In this module you are just telling toothpick how to create the object instance you want to use. You need to define it this way because the toothpick annotation processor can’t figure out how to build the instance of an interface on its own.

The other option is to create a class for your dependency without an interface. The benefit of doing it this way is that the tooling with Toothpick can pick up on this and provide instances of the dependency without any other configuration. You can still scope the dependency (for example to singleton) but you now have one file to maintain instead of two. This method is the preferred way to create your dependencies unless you have a complicated use case requirement or want to change out for a fake instance in your tests (for example, swapping retrofit’s base url configuration in unit tests to use OkHttp Mock Server).

Now that you have dependencies, you need to be able to use them (or else they are not dependencies at all, as noting would depend on them)! Toothpick supports two methods of injection: field and constructor. My recommendation, and many other developers, suggest always using constructor injection (there are some exceptions in Android components such as a View or Fragment which I will expand upon shortly). In Kotlin, this pattern is very clean and looks like this

{% highlight kotlin %}
class SomeDependency @Inject constructor(
    private val someOtherDependency: SomeOtherDependencyClass
) {
//... implement your class here ...
}
{% endhighlight %}

The tooling for toothpick will now create a factory class to build the dependency for you. If you inject a singleton, it will also make sure to inject the already created instance (or created one for you).  This also makes testing a bit easier, as you don’t need any library in your unit tests. If you use a library like mockito you can create the dependency very easily

{% highlight kotlin %}
val someOtherDependency: SomeOtherDependencyClass = mock()
val someDependency(someOtherDependency)
verify(someOtherDependency).someMethod()// make sure your mock was called
{% endhighlight %}

You can also use field injection as mentioned earlier. With this, you must create a public var and annotate it with `@Inject` to signal to the tooling that the factory should set the variables value.  The two major downsides here are that the variable is public and it is mutable, unlike the constructor invocation. This means a developer could accidentally set a new value for the injected field or some other class could access the dependency since it is a public variable with a getter and setter. You can’t get away from using this method entirely though, for fragments to be automatically recreated by the system you must keep the constructor empty. This means that a field must be used to get access to your dependency (most of the time). You could open a toothpick scope directly and get an instance of a variable, but this too has downsides

{% highlight kotlin %}
private val someOtherDependency: SomeOtherDependencyClass by lazy { Toothpick.openScope("scopeName").getInstance() }
{% endhighlight %}

With field injection, you need to tell toothpick you want to inject the fields. To do this, you call Toothpick.inject- until this method is called your fields are not initialized so if you use lateinit  you’ll get an exception and if you have a nullable type with null set, the variable might be null. For this reason, in android classes you typically need to handle the injection in onCreate before you call the super to insure the dependency gets injected.

Finally, the other concern of toothpick is its performance. In Toothpick3, reflection is used at runtime to load the dependencies via the generated factory methods. This is much slower than having compiled instructions at runtime, but is a bit faster at compile-time. There are other issues here too, like high memory use if you do not open and close your scopes at appropriate times (for example open scope onCreate and close the scope onDestroy), and many developers might be driven to creating singletons instead of actually managing scope.

## Hilt
Hilt is the new kid on the block at first glance, but when you dig in deeper you realize that Dagger 2 is the actual framework being used. Hilt is just a replacement for the previous (and very over-complicated) Dagger-Android. This also means that you can use Dagger2 ideas / modules with Hilt (with a few extra steps). The Hilt tooling is purposely built for Android with performance considerations on mobile platforms in mind. To get started, follow the hilt guide on getting started. Since this process is very well documented, I will not go into detail here outside of what troubles we ran into.

The setup is fairly straightforward, but did take a non-zero amount of time to complete in our environment. Firstly, you will want to look at all of the interfaces you have in your project thus far and determine if they are truly needed. If not, my recommendation is to remove the interface and rename the class to the name previously used by the interface. Once this is completed, you need to find all classes with injected fields and migrate them to constructor invocation. We are currently in the process of migration to Kotlin so many old classes were still in Java. Instead of doing the migration at the same time, we implemented constructor injection in Java like this

{% highlight java %}
public SomeClass {
    @NotNull
    private final SomeDependency someDependency;

    @Inject
    public SomeClass(
        SomeDependency someDependency
    ) {
        this.someDependency = someDependency;
    }
}
{% endhighlight %}

This process of migrating to constructing dependencies was much more time consuming than other steps, but our codebase is much cleaner and easier to test now thanks to this. 

Next, we needed to handle injecting our dependencies into android components. To support this, you need to add the annotation @HiltApplication to your Application class. If you currently do not override application, you can define this in the app/ module. If you take advantage of a multi module architecture, Hilt will still work as long as you are not using dynamic feature modules (if you are, you need to use dagger2 still, but Hilt is still in alpha so this may be supported in the future). In your other grade modules, you will include the hilt grade plugin and dependencies still. If you are providing an external library with a hilt module, you will need to make sure that you add that library as a dependency in the app/ grade module as well so that the code generated can reference those classes.

Other android components, such as View, Activity, Fragment,  and BroadcastReceiver are supported out of the box. If you use one of these components, add @AndroidEntryPoint to the class and the field injection will just work. The tooling will do a bytecode transformation to handle injection transparently.

If you have a component not supported by Hilt directly, not all hope is lost. We have some ViewHolder classes that weren’t easy to refactor right now, so for these we added a hacky method to create an entry point - we used itemView.context.applicationContext to access the application scope (and then our UseCase dependency which was already a singleton, meaning there were no performance implications of doing this). In the future, we would refactor these ViewHolders to draw off of a ViewState data class instead of doing complex logic at the view level (which is hard to test)

Finally, you are ready to migrate your modules. If you have dagger, you just need to add `@InstallIn(SomeComponent::class)` and you’re done. If moving from toothpick, you will likely need to create the modules but only if you need to provide an instance (like allowing injection of OkHttpClient, for example) or need to bind your interfaces to implementations.

Now you should be able to compile the app and run your code. If you run into issues here, you may have stumbled into a great feature of Dagger, compile-time correctness checks. Cyclic dependencies and scoping issues will be present at compile time, and are typically easy to fix with the error log in your build. Most of the time you just need to make sure objects do not depend on each other and refactor / brake up objects that do rely on each other. Logcat is also very helpful to determine how to fix runtime crashes if you run into them (such as trying to inject an entry point manually to the wrong scope).

Another great feature of Hilt is that it is supported directly by Google, meaning AndroidX is supported for a few components (currently WorkManager and ViewModel). With toothpick we were not able to easily access SavedStateHandle (which allows you to save state for when the system kills the app not just when you rotate the device or change theme). To add this support, read the Hilt documentation for supporting the Jetpack libraries (you need a new dependency and a new annotation processor). Then you need to annotate your ViewModel with `@ViewModelInject` and your SavedStateHandle with `@Assisted`. 

### Performance
This was the single largest technical win for our team in recent memory; our app now feels buttery smooth and is much more memory efficient. Part of this is thanks to removing  dependencies from the singleton scope that did not need to exist for the entire app lifecycle. The other part of this comes from Hilt destroying instances with the activity_fragment_view/etc now. Our development productivity has increased due to less overhead managing our dependencies and scopes and has made it easier for us to “do the right thing” with the android lifecycle (rotation, saving user state when background ed, etc). We still have more enhancements to make to our application, but Hilt has already shown vast improvements even in its alpha (functionally stable, APIs may change). Luckily Hilt is very simple to work with and most APIs are not used directly, so if they do change anything at the RC stage, migration will not be a burden on the team; even if it is a large migration, we still get to do the migration on our time as we elect when to upgrade the library.

## Bonus: Testing
All code you write is code you should test. If you are using Toothpick, Dagger, or Hilt testing will be supported and easier than manually injecting things and creating dependencies. 

Hilt’s mentality is that in your Unit Tests (that is, tests running on your development computer) you will use mocks and validate only functionality in that dependency, not all other dependencies. This makes sense as unit tests are supposed to be very fast and focused to allow rapid development and testing.

Alternatively, your instrumented tests should use real implementations as these are much slower tests which run on the android system - these are truly integration tests by definition. You can provide fakes for your configuration, like the URL used by retrofit so that you communicate with the mock server instead of the real api’s in your tests (which would make your tests flaky). The benefit of this approach is that you can create less unit tests but get more code coverage overall. The downside is that when tests inevitably break during development, it is harder to pin-point which part of the app is broken by a change.

In your instrumented tests, you will need to define a custom test runner and define it in your gradle configuration. This process is very well documented in the android developer documentation on Hilt.


Want to learn more? Feel free to reach out with any questions you may have, or read through some of these helpful documentation pages:

[Dagger Hilt Docs](https://dagger.dev/hilt/)

[Android Developer - Hilt](https://developer.android.com/training/dependency-injection/hilt-android)

[Android Developer - Hilt Testing](https://developer.android.com/training/dependency-injection/hilt-testing)


