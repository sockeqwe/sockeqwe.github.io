---
layout: post
published: true
title: Architecure Components - I'm not a purist but ...
mathjax: false
featrued: false
comments: true
headline: Architecure Components - I'm not a purist but ...
categories:
  - Android
tags: [android]
---
At I/O 2017 Google surprised us with a new initiative: [Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html). I really appreciate this initiative. In this blog post I would like to share my thoughts about ViewModel and some pitfalls you might stumble upon when using ViewModel and LiveData by taking a look at the official Google samples.

## The State Problem
It is not a secret that I'm a big fan of [Model-View-Intent (MVI)](http://hannesdorfmann.com/android/mosby3-mvi-1).
This is just a matter of personal preference one could argue and I'm not disagreeing.
I like MVI because state management, unidirectional data flow and immutability is a core part of the pattern.
On the other hand MVVM and MVP don't talk about state management although it can be (and should) applied in these patterns too.
For example Laimonas Turauskas from Instacart wrote a [blog post](https://tech.instacart.com/lce-modeling-data-loading-in-rxjava-b798ac98d80)
how they do state management in their MVP based app.
Fortunately, Google's MVVM samples show how to do that with ViewModel and LiveData too.
Take a look at [this example](https://github.com/googlesamples/android-architecture-components) which is an app that let's you browse Github. It's a very simplified Github client that uses Githubs REST API to display some data about git repositories and users in this app.

{% highlight java %}
public enum Status {
    SUCCESS,
    ERROR,
    LOADING
}

public class Resource<T> {

    public final Status status;
    public final String message;
    public final T data;

    public Resource(@NonNull Status status, @Nullable T data, @Nullable String message) {
        this.status = status;
        this.data = data;
        this.message = message;
    }

    public static <T> Resource<T> success(@Nullable T data) {
        return new Resource<>(SUCCESS, data, null);
    }

    public static <T> Resource<T> error(String msg, @Nullable T data) {
        return new Resource<>(ERROR, data, msg);
    }

    public static <T> Resource<T> loading(@Nullable T data) {
        return new Resource<>(LOADING, data, null);
    }
}
{% endhighlight %}

A **Resource** is used for example to wrap a http response i.e. **Resource&lt;ApiResponse&gt;** (where ApiResponse holds the json parsed data returned from github.com). Btw. if you use Kotlin, you should take a look at **sealed classes**.

I'm super happy to see such state management related code in official Google samples and
encourage developers to build your apps with proper state management in mind.
However, in the same Google example we can scroll through a infinite list of search results (uses pagination). Some interesting observation:

{% highlight java %}
public class SearchViewModel extends ViewModel {
  ...
  public LiveData<Resource<List<Repo>>> getResults() { ... }
  public LiveData<LoadMoreState> getLoadMoreStatus() { ... }
}
{% endhighlight %}

**getResults()** is used to represent the search results whereas **getLoadingMoreStatus()** is used for consecutive pagingation.

Why not just one **LiveData&lt;State&gt; getState()**? Wouldn't it make state management especially in a multi threaded environment more predictable and deterministic?

I'm not a purist but it leaves a stale after-taste ...

## The mantra problem
**LiveData** seems to be a very simplified version of RxJava's **Observable**.
LiveData is lifecycle aware so that we as developer don't have to unsubscribe explicitly as we have to do with RxJava's Observable. Both implement the Observer pattern.
LiveData doesn't provide all this fancy functional programming operators as RxJava does although architecture components provides some functional concepts via **Transformations** helper class such as **Transformations.map()** and **Transformations.swithMap()**.

Quite often I hear developers saying how much they like functional reactive programming and pure functions without side effects but then we write code like **someLiveData.setValue(...)**.

I'm not a purist but it leaves a stale after-taste ...

## The state restoration race
[Rebecca Franks](https://twitter.com/riggaroo) who seems to be working on a sample app
(with LiveData and Architecture Components) asked an interesting question in a public slack chat about a Google TODO app example, more specific about the screen where the user can create a new TODO Task:  

> Is there a reason why the Fragment passes information back to the ViewModel when calling **saveTask(title, description)**. The ViewModel has the title and description already, so does that need to be passed into the saveTask(title, description) method explicitly?

The code she is referring to can be found [here](https://github.com/googlesamples/android-architecture/blob/dev-todo-mvvm-live/todoapp/app/src/main/java/com/example/android/architecture/blueprints/todoapp/addedittask/AddEditTaskFragment.java#L111).
At first glance the answer seems to be obvious: _"No, there is no reason to pass that information back from View to the ViewModel because the ViewModel already holds this information"_.
Is that always true?

Well, ViewModel is "only" kept across screen orientation changes.
On the other hand UI widgets like EditText store and restore their own state.
Do you see the conflict potential?
Just for fun, let's also add android data binding into this equation and let me quote [Yigit Boyar](https://twitter.com/yigitboyar):

> There is one issue with data binding though. Data binding does not work well with state restoration since it expects to own the views. So data binding will go and override views. It is very annoying but there is no easy solution for that.

I'm not really answering  Rebecca's question because the previous answer that has seemed to be so obvious is actually not that obvious anymore. So which one is the source of truth? Should the ViewModel's state be persisted too (see [this post](https://proandroiddev.com/customizing-the-new-viewmodel-cf28b8a7c5fc) by [Danny Preussler](https://twitter.com/PreusslerBerlin))? Which one wins the state restoration race?

I'm not a purist but it leaves a stale after-taste ...

## The "SingleLiveEvent" problem
[Fabio Collini](https://twitter.com/fabiocollini) raised an interesting [question](https://github.com/googlesamples/android-architecture-components/issues/63) about the Google's sample github client app that is powered by the new Architecture Components:

> Sometimes the ViewModel needs to invoke an action using an Activity (for example to show a snack bar or to navigate to a new activity). Using MVP it's easy to implement it because the presenter has a reference to the Activity. The ViewModel doesn't contain a reference to the Activity, what's the best way to implement this feature?

What he means is an "action" that takes place only one time like displaying a SnackBar or Toast.
But if you would model this problem in MVVM you would have a **LiveDataa&lt;Stringa&gt; getSnackBarErrorMessage()**  in your ViewModel and the View subscribes to it.
If the emitted String (error message) is null, don't display a SnackBar, otherwise display it.
The problem he is facing is that this flag must be cleared somehow
otherwise the SnackBar is shown again after a screen orientation change because LiveData is emitting the latest (cached) value when the view (re)subscribes to it.

In the Google samples they have added a class called [SingleLiveEvent](https://github.com/googlesamples/android-architecture/blob/dev-todo-mvvm-live/todoapp/app/src/main/java/com/example/android/architecture/blueprints/todoapp/SingleLiveEvent.java).
The idea is that once an event has been dispatched it sets the internal value to null.
This prevents the SnackBar to appear a second time after screen orientation change since the error message string has been set internally to null.

At the same time developer are arguing that we should focus on immutability but guess what:
SingleLiveEvent is not immutable and also has side effects in it (setting the value to null).
Well, SingleLiveEvent seems to solve this problem, but isn't it just hiding the real underlying problem (hint: state management)?
On the other hand, at the end of the day a user of the app doesn't care about immutability as long as the app works as expected. So shouldn't we developer don't care about immutability too as long as we get the job done somehow?
Btw. I have commented on that issue too and suggested an [alternative solution](https://github.com/googlesamples/android-architecture-components/issues/63#issuecomment-310422475)

I'm not a purist but it leaves a stale after-taste ...


## Conclusion: I'm a purist
As you see, even in such small sample apps like a Github client or a TODO app we face some tricky edge cases.
From my point of view this is not related to any architectural pattern of our choice.
It doesn't matter if you prefer MVVM with the new Architecure Components or MVP or MVI or whatever else.
This kind of problems are just the symptoms of a much deeper issue we face but maybe haven't thought about it yet: proper state management.

Google's Architecture Components are awesome and this blog post is by no means a critique on that library nor on the examples Google (and many other external contributors) provided us.
I just think that with the introduction of Architecture Components we (including Android Developers from around the world and Google Developers) have a very unique chance
to create examples that are a great starting point for any new Junior Android developer by agreeing on the importance of state management regardless if you are a purist or not.
Finally, this would help a lot developers avoiding pain points in the future from lessons we have learned the hard way in the past. Let's reflect that in official Google examples. They are open source, let's contribute!