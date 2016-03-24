---
layout: post
published: true
title: Presenters don't need lifecycle events
mathjax: false
featured: false
comments: true
headline: Presenters don't need lifecycle events
categories:
  - Android
tags: [android, software-architecture, design-patterns]
---

Lately I have been asked why Presenters in Mosby (MVP library) don't have lifecycle callback methods like onCreate(Bundle), onResume() etc. Also the awesome guys over at SoundCloud have published a library called [LightCycle](https://github.com/soundcloud/lightcycle) that helps break logic out of Activity or Fragments into smaller containers bound to the parents Activity's or Fragment's lifecycle. While this library is great and helpful they also mention in their [examples](https://github.com/soundcloud/lightcycle/blob/master/examples/real-world/src/main/java/com/soundcloud/lightcycle/sample/real_world/HeaderPresenter.java) that this library can be used in MVP to bring lifecycle to Presenters. I, personally, think that Presenters don't need lifecycle callback methods and in this blog post I will discuss why.

Alright, before we get started, think of the definition and the rule of a `Presenter` in MVP. I will give you my definition:

> The Presenter is responsible to coordinate the view, transform Model into "PresentationModel" so that the View can display data more convenient and to be the bridge to the "business logic" to retrieve data that should be displayed in the View. MVP is all about separation of concerns.

Given that "definition" of a Presenter I don't see a reason why the Presenter needs lifecycle callbacks like `onCreate()`, `onPause()` and `onResume()`. All the Presenter needs to know is whether or not a View is attached to the Presenter (Hence [Mosby Presenter](https://github.com/sockeqwe/mosby/blob/master/mvp-common/src/main/java/com/hannesdorfmann/mosby/mvp/MvpPresenter.java) only have `attach()` and `detach()` methods if you want to count them as "lifecycle events"). It should be that simple. Actually, this is one of the arguments why developers are advocating against fragments (complex lifecycle) and prefer "custom views" (i.e. extending from FrameLayout) because custom view's only have two lifecycle events: ViewGroup.onAttachedToWindow() and ViewGroup.onDetachedFromWindow().

Furthermore, lifecycle managing is already a complex topic. If you start to move lifecycle logic in your Presenter, then you basically have just moved that spaghetti code from Activity to Presenter. I think there is the need of a separated component responsible for lifecycle handling, the Activity, and not the Presenter. Clearly the intention of  SoundCloud's [LightCycle](https://github.com/soundcloud/lightcycle) is not to move all that code from Activity to Presenter. In fact they want to split or delegate the spaghetti code logic you usually write in your Activity's or Fragment's lifecycle methods into multiple components to build smaller components following the [single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle). Don't get me wrong: The idea is great and achieving that via annotation processing is pretty smart, but it's not the Presenters responsibility and therefore the Presenter doesn't need lifecycle callback methods.

With that said, you might understand my point of view, but you also have faced scenarios where from your point of view the Presenter indeed needs lifecycle callback methods. Okay, let's talk about it!

For example let's assume we want to build an app that displays your current position on a map in your app. As the screen goes off you want to stop retrieving GPS position updates to safe battery life. So how would we implement this with MVP? You would have an `TrackingActivity` as View (`TrackingView`) displaying a map with the user's current GPS position. Also you need a `GpsTracker` class as business logic" (model in MVP) which is responsible to detect the current GPS position and notify observers (listeners) about position changes. The `TrackingPresenter` is such an observer. He is registered as listener to `GpsTracker` and will update the View when gps position has been changed.

So far so good. As already said, to not drain the battery too much we want to stop GPS tracking when the display is off, in other words stop GPS Tracking in `Activity.onPause()` and resume when `Activity.onResume()`. So again, it's not the Presenter who needs onPause() and onResume() lifecycle events but rather the "business logic", the `GpsTracker`, needs these lifecycle events.

But how do we implement that? Should we simply forward `Activity.onPause()` to `Presenter.onPause()` which then calls `GpsTracker.stop()`? So we do need a Presenter with lifecylce methods otherwise we couldn't forward the pause events the way down to `GpsTracker.stop()`, right?
I think there is a better way. As already said, is not the responsibility of the presenter to handle lifecycle events. Actually, there is already a component that is responsible for lifecycle events: the Activity (or Fragment). So instead of forwarding Activity.onPause() and Activity.onResume() to the presenter, just do something like this:

```java
class TrackingActivity extends MvpActivity implements TrackingView {
     private GpsTracker tracker;

     public onCreate(){
            tracker = new GpsTracker(this); // might need a context
            ...
     }

     public void onPause(){
         tracker.stop();
     }

    public void onResume(){
         tracker.start();
     }

    @Override
    public void createPresenter(){ // Called by Mosby
          return new TrackingPresenter(tracker);
     }
}
```

As you see, the trick is that the `GpsTracker` is "bound" to the Activity's lifecycle directly in the component that is responsible for managing lifecycle events: the Activity! Then we pass the GpsTracker to the Presenter as constructor parameter.
Furthermore, now `TrackingPresenter` fulfills the single responsibility from my previous definition: It's only responsible to update the View.

```java
class TrackingPresenter extends MvpBasePresenter<TrackingView>
                        implements GpsUpdateListener{

  GpsTracker tracker;

  public TrackingPresenter(GpsTracker tracker){
    this.tracker = tracker;
    tracker.setGpsUpdateListener(this);
  }


  @Override
  public void onGpsLocationUpdated(GpsPosition postion){
    view.showCurrentPosition(postion.getLat(), position.getLng());
  }
}
```

That's it. **Single responsibility for all your components!**

Of course we can spice up the code shown above with dependency injection and with LightCycle:

```java
class TrackingActivity extends MvpActivity implements TrackingView {
    @Inject @LightCycle GpsTracker tracker;

    @Override
    public TrackingPresenter createPresenter(){
          return getObjectGraph.get(TrackingPresenter.class); // Dagger 1
          // or
          return getComponent().trackingPresenter(); // Dagger 2
     }
}
```

Are you looking for more examples? Another user of Mosby (MVP library) have asked me a similar question on github with a music player app he is working on. I gave a similar answer [there](https://github.com/sockeqwe/mosby/issues/124)

Of course this is only my personal opinion and there are exceptions and you may really need lifecylce events in your Presenters, but I think in 99% your business logic needs lifecylce events and not your Presenters.

Last but not least I want to say that the intention of this blog post is not to discredit SoundCloud's LigthCycle! I just wanted to say that from my point of view the LightCycle [examples](https://github.com/soundcloud/lightcycle/blob/master/examples/real-world/src/main/java/com/soundcloud/lightcycle/sample/real_world/HeaderPresenter.java) (having Presenters with lifecycle callback methods) are not my cup of tea. Moreover, I think this is caused by the fact that my definition of a Presenter (i.e. I also don't want to have android SDK dependencies like `Bundle` in my Presenters) is entirely different from SoundClouds definition of a Presenter.