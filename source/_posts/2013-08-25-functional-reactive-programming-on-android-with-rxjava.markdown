---
layout: post
title: "Functional Reactive Programming on Android with RxJava"
date: 2013-08-25 09:05
comments: true
categories:
- android
- rxjava 
---

If you are an application developer, there are two inconvenient truths:

1. Modern applications are inherently concurrent.
1. Writing concurrent programs that are correct is difficult.

In the domain of mobile or desktop applications,
parallel execution allows for responsive user interfaces because we can move
computations into the background while the UI responds to ongoing
user interactions. Code must execute concurrently
to not stray from this fundamental requirement. 
Writing such programs
is diffcult because on mobile they are typically written in imperative
languages like C or Java. Writing concurrent code in imperative
languages is difficult because code is written in terms of interweaved,
temporal instructions that move objects
or data structures from one state to another. This imperative style of programming 
inherently produces side effects. It presents several problems when running 
instructions in parallel, such as race conditions when writing to a shared resource.

# Resistance is futile--or is it?

Developers have grown accustomed to the drawbacks of
expressing concurrency in imperative languages.
On platforms like Android where Java is (still) the
dominant language, concurrency simply sucks, and we should just give in and deal with it.
I personally keep a close eye on the server side end of the spectrum. 
Over the past few years,
functional programming has made an astounding comeback 
in terms of rate of adoption and innovation, the details of which 
I will not get into here. In the case of concurrency, functional programming has
a very simple answer to dealing with shared state: don't have it.

## Problems of concurrent programming with AsyncTask

Being based on Java, Android comes with a standard number of Java concurrency
primitives such as `Thread`s and `Future`s. While these tools make 
it easy to perform simple asynchronous tasks, they are fairly low level and require
a substantial amount of diligence when you use them to model complex
interactions between concurrent objects. A frequent use case on Android
or any UI-driven application is to perform a background job
and then update the UI with the result of the operation. Android provides `AsyncTask`
for exactly that:

{% codeblock lang:java %}
class DownloadTask extends AsyncTask<String, Void, File> {
 
  protected File doInBackground(String... args) {
    final String url = args[0];
    try {
      byte[] fileContent = downloadFile(url);
      File file = writeToFile(fileContent);
      return file;
    } catch (Exception e) {
      // ???
    }
  }
  
  protected void onPostExecute(File file) {
    Context context = getContext(); // ???
    Toast.makeText(context, 
        "Downloaded: " + file.getAbsolutePath(), 
        Toast.LENGTH_SHORT)
        .show();
  }
}
{% endcodeblock %}

This looks straight forward. We define a method `doInBackground` which accepts
something through its formal parameters, and returns something as the result
of the operation. Android guarantees us that this code will execute in a thread
that's not the main user interface thread. We also define a UI callback called
`onPostExecute`, which receives the result of the computation and can consume
it on the main UI thread, since Android guarantees that this method will always
be invoked on the main thread. So far so good. What's wrong with this picture?

### In search for those jigsaw pieces

Quite a few things as it turns out. Let's start with `doInBackground`. As you
can see we download a file here, a costly operation since it involves network and disk I/O.
There's about a hundred things that can go wrong here, so we surely want to
recover from errors, and add a try-catch block. Just what do we do in the catch
block? Log the error? That's good and fine, but perhaps we may want to inform
the user about this error too, which likely involves interacting with the UI, but
wait, we can't do that sine we're not allowed to update user interface elements
from a background thread. Bummer.

Surely it should be easy to handle that error in `onPostExecute` then.
We might reason that it's as simple as holding on to the exception in a private
field (i.e. we write it on the background thread), and check in `onPostExecute` 
(i.e. read it on the UI thread) if that field is set to something other than
null (did I mention we love null checks) and display it to the user in some way
shape or form. (The observant reader at this point notes that text in parens
equals: subtle problems ahead.) But wait, how do we obtain a reference to a
`Context`, without which we cannot do anything meaningful with the UI? Apparently
we have to bind it to the task instance up front, at the point of instantiation,
and keep a reference to it throughout the task execution. But what if the download
takes a minute to run? Do we want to hold on to e.g. an `Activity` instance
for an entire minute? What if the user decides to back out of the Activity which
triggered the task, and we're now holding on to a stale reference, which not only
creates a substantial memory leak, but also isn't worth a dime, since it
has meanwhile been detached from the application window? A problem that, as [a 
simple Google search suggests](https://www.google.de/search?q=asynctask+configuration+change), everyone is well aware of.

### Beyond the basics

Without wanting to beat horses that are already dead, there are other problems
with all this. The task above is incredibly simple. Picture more complicated
scenarios where we need to orchestrate a number of such operations. We might
for instance want to fetch some JSON from a service API, parse it, map it,
filter it, cache it to disk, and only then feed the result to the UI. All
the operations just mentioned should--as per the single responsibility principle--exist
as separate objects, perhaps exposed through different services. Composing
these using `AsyncTask` is difficult and non-intuitive, since it requires
grouping any number of combinations of service interactions into separate task
classes, resulting in a proliferation of--from the perspective of your business
logic--meaningless task classes.

Another option would be to have one task class per service object invocation, or
have the service objects themselves wrap their code in `AsyncTasks`. This then
means that composing service objects means nesting `AsyncTask`, leading to what's
commonly called the "callback hell", since you start tasks from a task callback
from a task callback from... you get the idea.

Last, but not least, `AsyncTask`s scheduling behavior varies significantly across
different versions of Android. I believe I have seen it changing from a capped
thread pool back in the 1.x days (with varying bounds depending again on the API level) 
to a _single thread executor_ model in 4.x. Read that again: your tasks (plural) 
do _not_ run concurrently to each other on ICS devices and beyond (although they do run
concurrently to the main UI thread). Why did GOOG decide to serialize task execution?
For the same reason I'm writing this post. Developers simply couldn't get it right,
applications suffered from nasty problems due to race conditions and incorrectly
synchronized code.

### The inconvenient truth

So, should we still use `Thread` and `AsyncTask`?

The answer is: probably. For very simple, one-shot jobs that do not require
much orchestration, `AsyncTask` is fine. For anything beyond that:
it's doable, but it requires juggling with `volatile`s, `WeakReference`s,
`null` checks, and other [defensive, unconfident mechanisms](http://devblog.avdi.org/2012/06/05/confident-ruby-beta/), 
but perhaps worst of all, it requires us to
think about things that have nothing to do with the problem we set out to solve: 
downloading a file.

# Enter RxJava--now with more Android

To come back to the initial problem statement: do we have to give in to the lack
of high level abstractions and deal with it, or are there better solutions? Turns
out that as initially hinted at, functional programming might have an answer to
this. "But wait" you might say, "I still want to use Java?". Turns out: yes we can.
It's not super pretty (at least not unless Google whip out their magic wand and
give us Java 8 and closures on Dalvik, or unless you feel attracted to anonymous classes and
six levels of identation), but it happens to solve all of the above problems in
one fell swoop. To summarize again what these problems were:

- No standard mechanism to recover from errors
- Lack of control over thread scheduling (unless you like to dig deep)
- No obvious way to compose asynchronous operations
- No obvious and hassle free way of attaching to `Context`

[RxJava](https://github.com/Netflix/RxJava) is an implementation of the Reactive Extensions (Rx)
on the JVM, courtesy of Netflix. Rx was first conceived by Erik Meijer
on the Microsoft .NET platform, as a way of combining data or event streams with
reactive objects and functional composition. In Rx, events are modeled
as observable streams, to which observers are subscribed. These streams, or observables
for short, can be filtered, transformed and composed in various ways before their
results are emitted to an observer. Every observer is defined over three simple messages:
`onNext`, `onCompleted`, and `onError`. Concurrency is a variable in this equation,
and abstracted away in the form of schedulers. Generally, every observable stream
exposes an interface that is modeled around concurrent execution flows (i.e. you don't
call it, you subscribe to it), but by default is executed synchronously. Introducing
schedulers can then make an observable execute using various concurrency primitives
such as threads, thread pools, or even Scala actors. Here's an example:

{% codeblock lang:java %}
Subscription sub = Observable.from(1, 2, 3, 4, 5)
    .subscribeOn(Schedulers.newThread())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(observer);

// ...

sub.unsubscribe();
{% endcodeblock %}

This creates a new observable stream from the given list of integers, and emits
them one after another on the given observer. The use of `subscribeOn` and `observeOn`
configures the stream to emit the numbers on a new `Thread`, and receive them on
the Android main UI thread (i.e. the observer's `onNext` method is called on the main thread.)
Eventually, we also unsubscribe from the observable.

Here's a possible `Observer` implementation:

{% codeblock lang:java %}
public class IntObserver implements Observer<Integer> {
 
  public void onNext(Integer value) {
     System.out.println("received: " + value);
  }

  // onCompleted and onError omitted
  ...
}
{% endcodeblock %}

Now this is not a very interesting example. Let's have a look at how we might implement
the download task as an Rx `Observable`.

{% codeblock lang:java %}
private Observable<File> downloadFileObservable() {
    return Observable.create(new Func1<Observer<File>, Subscription>() {
        @Override
        public Subscription call(Observer<File> fileObserver) {
            try {
                byte[] fileContent = downloadFile();
                File file = writeToFile(fileContent);
                fileObserver.onNext(file);
                fileObserver.onCompleted();                    
            } catch (Exception e) {
                fileObserver.onError(e);
            }
            return Subscriptions.empty();
        }
    });
}
{% endcodeblock %}

What we do here is create a method which builds an `Observable` "stream" (in our case emitting
only ever a single item, the file), to which a `File` observer can connect. Whenever
this observable is subscribed to, its `call` function will trigger and execute the task at hand.
If the task can be carried out successfully, we deliver the result to the observer through `onNext`,
so it can properly react to it. We then signal completion through `onCompleted`. If an exception
is raised, we deliver it to the observer through `onError`. Here is how you use this
from, say, a `Fragment`:

{% codeblock lang:java %}
class MyFragment extends Fragment implements Observer<File> {
  private Subscription subscription;

  protected void onCreate(Bundle savedInstanceState) {
    subscription = downloadFileObservable()
                          .subscribeOn(Schedulers.newThread())
                          .observeOn(AndroidSchedulers.mainThread())
                          .subscribe(this);
  }

  protected void onDestroy() {
    subscription.unsubscribe();
  }

  public void onNext(File file) {
    Toast.makeText(getActivity(),
        "Downloaded: " + file.getAbsolutePath(), 
        Toast.LENGTH_SHORT)
        .show();    
  }

  public void onCompleted() {}

  public void onError(Throwable error) {
    Toast.makeText(getActivity(),
        "Download failed: " + error.getMessage(), 
        Toast.LENGTH_SHORT)
        .show();    
  }
}
{% endcodeblock %}

As you can see, with RxJava the aforementioned issues have been solved in one fell swoop.
We can have proper error handling through an observer's `onError` callback. We can
execute the task on any given scheduler with a simple method call, and have fine grained
control over where the expensive code is run and where the callbacks will run, without
the need to write a single line of synchronization logic. RxJava also allows us to
compose and transform observables to obtain new ones, thus enabling easy reuse of code.
For instance, in order to not emit the `File` itself, but merely its path, we can take
the _existing_ observable and transform it:

{% codeblock lang:java %}
Observable<String> filePathObservable = downloadFileObservable().map(new Func1<File, String>() {
    @Override
    public String call(File file) {
        return file.getAbsolutePath();
    }
});

// now emits file paths, not `File`s
subscription = filePathObservable.subscribe(/* Observer<String> */);
{% endcodeblock %}

It should be easy to see how powerful this way of expressing asynchronous computations is.
At [SoundCloud](https://soundcloud.com) we are currently in the process of moving most of our code that relies
heavily on event based and asynchronous operations to Rx observables. For everyone's
convenience, for RxJava 0.10.1 we have contributed `AndroidSchedulers` which schedule an observer to receive callbacks
on a `Handler` thread (see [rxjava-android](https://github.com/Netflix/RxJava/tree/master/rxjava-contrib/rxjava-android)). We also plan to open source more
components that we currently use, but that are still being refined. For instance, we use
a custom `Observer` which makes guarantees that callbacks to fragments only happen
whenever the fragment is attached and the Activity is alive, thus making your own code
more reliable.

In a nutshell, RxJava finally makes concurrency and event based programming on Android hassle free.
It should be noted that we follow the same strategy on iOS using GitHub's [Reactive Cocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)
library, since as a team we have committed ourselves to the functional reactive paradigm.
We think it's an exciting development that leads to more stable code that is easy to unit test,
and free of low level state or concurrency concerns that would otherwise take over your
service objects.

If you're interested to hear more about this topic, make sure to watch this [interview
with our Director of Mobile Engineering on Root Access Berlin](http://backstage.soundcloud.com/2013/08/responsive-android-applications-with-sane-code/)
and come see me at [DroidCon UK 2013](http://uk.droidcon.com/2013/lineup/), where I will speak about RxJava and its use in
the SoundCloud application on the developer track.
