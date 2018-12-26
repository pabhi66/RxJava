# Rx Java

## The Basics

The building blocks of Rx are `Observables` and `Subscribers`. An Observable emits items and a Subscriber consumes those items.

There is a pattern to how items are emitted. An `Observable` may emit any number of items, then it terminates either by successfully completing, or due to an error. For each `Subscriber` it has, an Observable calls `Subscriber.onNext()` any number of times, followed by either `Subscriber.onComplete()` or `Subscriber.onError()`.

Observers often do not start emitting until someone explicitly subscribes to them. I.e. if no one is there to listen, the tree it won't emit items.

## Hello, World

```Java
Observable<String> myObservable = Observable.create(
    new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> sub) {
            sub.onNext("Hello, World!");
            sub.onCompleted();
        }
    }
);
```

Our `Observable` emits "Hello, World!" then completes. Now lets create a `Subscriber` to consume the data:

```Java
    Subscriber<String> mySubscriber = new Subscriber<String>() {
        @Override
        public void onNext(String s) {
            System.out.println(s);
        }

        @Override
        public void onComplete() {}

        @Override
        public void onError(Throwable e) {}
    }
```

All this does is print each String emitted by `Observable`.

We can  hook them up using `Subscribe()`:

```Java
    myObservable.subscrive(mySubscriber)
    // outputs "Hello, World!"
```

## Simpler code

```Java
    Observable<String> myObservable = Observable.just("Hello, world!");
```

- `Observable.just()` emits a single item then completes.

```Java
    Observable.just("Hello, world!")
        .subscribe(s -> System.out.println(s));
```

## Transformation

Suppose I want to append my signature to the "Hello, world!" output.

```Java
    Observable.just("Hello, world!")
        .subscribe(s -> System.out.println(s + " -Abhi"));
```

## Operators

Operators can be used in between the source `Observable` and the ultimate `Subscriber` to manipulate emitted items. Rx Java comes with many operartors such as `Map`, `FlatMap`, `Buffer`, `GroupBy`, `Scan`, `Window`.

## Map

`Map` operator can be used to transform one emitted item into another:

```Java
Observable.just("Hello, world!")
    .map(s -> s + " -Abhi")
    .subscribe(s -> System.out.println(s));
```

Map does not have to emit the items of the same type as Observable.

```Java
// outputing hashcode instead of the string "Hello, world!"
Observable.just("Hello, world!")
    .map(s -> s.hashCode())
    .map(i -> Integer.toString(i))
    .subscribe(s -> System.out.println(s));
```

Key idea #1: Observable and Subscriber can do anything.

- Your Observable could be a database query, the Subscriber taking the results and displaying them on the screen. Your Observable could be a click on the screen, the Subscriber reacting to it. Your Observable could be a stream of bytes read from the internet, the Subscriber could write it to the disk.

Key idea #2: The Observable and Subscriber are independent of the transformational steps in between them.

- I can stick as many map() calls as I want in between the original source Observable and its ultimate Subscriber. The system is highly composable: it is easy to manipulate the data. As long as the operators work with the correct input/output data I could make a chain that goes on forever4.

- Combine our two key ideas and you can see a system with a lot of potential. At this point, though, we only have a single operator, map(), which severely limits our capabilities. In part 2 we'll take a dip into the large pool of operators available to you when using RxJava.

## Flatmap

`Observable.flatMap()` takes the emission of one `Observable` and returns the emission of another `Observable` to take its place.

```Java
//I want to make a robust system for searching text and displaying the results.

// Returns a List of website URLs based on a text search
Observable<List<String>> query(String text);

query("Hello, world!")
    .flatMap(urls -> Observable.from(urls))
    .subscribe(url -> System.out.println(url));
```

The key concept here is that the new Observable returned is what the Subscriber sees. It doesn't receive a List<String> - it gets a series of individual Strings as returned by Observable.from().

- Suppose I have this method

```Java
    // Returns the title of a website, or null if 404
    Observable<String> getTitle(String URL);
```

- Now instead of printing the URLs , now I want to print the title of each website received. We can do this with flatmap(). After spliting the list of URLs into individual items, we can use getTitle() in flatmap() for each url before it reaches the Subscriber.

```Java
    query("Hello, world!")
        .flatMap(urls -> Observable.from(urls))
        .flatMap(url -> getTitle(url))
        .subscribe(title -> System.out.println(title));
```

- We are now composing multiple independent mehtods returning Observables together. We are combining two API calls into a single chain. We could do this for any number of API calls.

### `filter()`

emits the same item it receives, but only if it passes the boolean check.

```Java
    // don't print title if its value is null.
    query("Hello, world!")
        .flatMap(urls -> Observable.from(urls))
        .flatMap(url -> getTitle(url))
        .filter(title -> title != null)
        .subscribe(title -> System.out.println(title));
```

### `take()`

emits, at most the number of items specified. If there are fewer than 5 titles it'll just stop early.

```Java
    // print first 5 items
    query("Hello, world!")
        .flatMap(urls -> Observable.from(urls))
        .flatMap(url -> getTitle(url))
        .filter(title -> title != null)
        .take(5)
        .subscribe(title -> System.out.println(title));
```

### `doOnNext()`

allows us to add extra behavior each time an item is emitted.

```Java
    // save title
    query("Hello, world!")
        .flatMap(urls -> Observable.from(urls))
        .flatMap(url -> getTitle(url))
        .filter(title -> title != null)
        .take(5)
        .doOnNext(title -> saveTitle(title))
        .subscribe(title -> System.out.println(title));
```

## Error Handling

```Java
Observable.just("Hello, world!")
    .map(s -> potentialException(s))
    .map(s -> anotherPotentialException(s))
    .subscribe(new Subscriber<String>() {
        @Override
        public void onNext(String s) { System.out.println(s); }

        @Override
        public void onCompleted() { System.out.println("Completed!"); }

        @Override
        public void onError(Throwable e) { System.out.println("Ouch!"); }
    });
```

Lets say potentialException() and anotherPotentialException() both have the possibility of thwoing Excecption. Then The output of the program will either be a String followed by "Completed!" or it will just be "Ouch!" (because an Exception is thrown).

- `onError()` is called when exception is thrown at anytime. Making it easier to handle every errors in a single function.

- The operator does not have to handel the error. You can leave it up to the Subscriber to determine how to handle issues with any part of the Observable chain because Exceptions skip ahead to onError().

## Schedulaers

You've got an Android app that makes a network request. That could take a long time, so you load it in another thread. Suddenly, you've got problems!

Multi-threaded Android applications are difficult because you have to make sure to run the right code on the right thread; mess up and your app can crash. The classic exception occurs when you try to modify a View off of the main thread.

- In RxJava, you can tell your Observable code which thread to run on using `subscribeOn()`, and which thread your Subscriber should run on using `observeOn()`:

```Java
myObservableServices.retrieveImage(url)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(bitmap -> myImageView.setImageBitmap(bitmap));
```

- With an `AsyncTask` or the like, I have to design my code around which parts of the code I want to run concurrently. With `RxJava`, my code stays the same - it's just got a touch of concurrency added on.

## Subscription

- When you call `Observable.subscribe()`, it returns a `subscription`. This represents a link between `Observable` and `Subscriber`.

```Java
Subscription subscription = Observable.just("Hello, World!")
    .subscribe(s -> System.out.println(s));
```

You can use this Subscription to sever the link later on:

```Java
subscription.unsubscribe();
System.out.println("Unsubscribed=" + subscription.isUnsubscribed());
```

What's nice about how RxJava handles unsubscribing is that it stops the chain. If you've got a complex chain of operators, using unsubscribe will terminate wherever it is currently executing code. No unnecessary work needs to be done!

---

## `RxAndroid`

`RxAndroid` is an extension to RxJava built just for Android. It includes special bindings that will make your life easier.

First, there's `AndroidSchedulers` which provides schedulers ready-made for Android's threading system. Need to run some code on the UI thread? No problem - just use `AndroidSchedulers.mainThread()`:

**Note** `AndroidSchedulers.mainThread()` uses a `HandlerThreadScheduler` internally, in fact.

```Java
retrofitService.getImage(url)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(bitmap -> myImageView.setImageBitmap(bitmap));
```

- Next we have `AndroidObservable` which provides more facilities for working within the Android lifecycle. There is `bindActivity()` and `bindFragment()` which, in addition to automatically using `AndroidSchedulers.mainThread()` for observing, will also stop emitting items when your Activity or Fragment is finishing (so you don't accidentally try to change state after it is valid to do so).

```Java
AndroidObservable.bindActivity(this, retrofitService.getImage(url))
    .subscribeOn(Schedulers.io())
    .subscribe(bitmap -> myImageView.setImageBitmap(bitmap));
```

- `AndroidObservable.fromBroadcast()`, which allows you to create an Observable that works like a BroadcastReceiver. Here's a way to be notified whenever network connectivity changes:

```Java
IntentFilter filter = new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION);
AndroidObservable.fromBroadcast(context, filter)
    .subscribe(intent -> handleConnectivityChange(intent));
```

- `ViewObservable`, which adds a couple bindings for Views. There's `ViewObservable.clicks()` if you want to get an event each time a View is clicked, or `ViewObservable.text()` to observe whenever a TextView changes its content.

```Java
ViewObservable.clicks(mCardNameEditText, false)
    .subscribe(view -> handleClick(view));
```

## Retrofit

```Java
@GET("/user/{id}/photo")
Observable<Photo> getUserPhoto(@Path("id") int id);

Observable.zip(
    service.getUserPhoto(id),
    service.getPhotoMetadata(id),
    (photo, metadata) -> createPhotoWithData(photo, metadata))
    .subscribe(photoWithData -> showPhoto(photoWithData));
```

## Lifecycle

- Continuing a Subscription during a configuration change (e.g. rotation). Suppose you make REST call with Retrofit and then want to display the outcome in a ListView. What if the user rotates the screen? You want to continue the same request, but how?

- Memory leaks caused by Observables which retain a copy of the Context. This problem is caused by creating a subscription that retains the Context somehow, which is not difficult when you're interacting with Views! If Observable doesn't complete on time, you may end up retaining a lot of extra memory.

** The first problem can be solved with some of RxJava's built-in caching mechanisms, so that you can unsubscribe/resubscribe to the same Observable without it duplicating its work. In particular, `cache()` (or replay()) will continue the underlying request (even if you unsubscribe). That means you can resume with a new subscription after Activity recreation:

```Java
Observable<Photo> request = service.getUserPhoto(id).cache();
Subscription sub = request.subscribe(photo -> handleUserPhoto(photo));

// ...When the Activity is being recreated...
sub.unsubscribe();

// ...Once the Activity is recreated...
request.subscribe(photo -> handleUserPhoto(photo));
```

**Note**: that we're using the same cached request in both cases; that way the underlying call only happens once. Where you store request I leave up to you, but like all lifecycle solutions, it must be stored somewhere outside the lifecycle (a retained fragment, a singleton, etc).

** The second problem can be solved by properly unsubscribing from your subscriptions in accordance with the lifecycle. It's a common pattern to use a CompositeSubscription to hold all of your Subscriptions, and then unsubscribe all at once in `onDestroy()` or `onDestroyView()`:

```Java
private CompositeSubscription mCompositeSubscription
    = new CompositeSubscription();

private void doSomething() {
    mCompositeSubscription.add(
        AndroidObservable.bindActivity(this, Observable.just("Hello, World!"))
        .subscribe(s -> System.out.println(s)));
}

@Override
protected void onDestroy() {
    super.onDestroy();
    mCompositeSubscription.unsubscribe();
}
```

For bonus points you can create a root Activity/Fragment that comes with a CompositeSubscription that you can add to and is later automatically unsubscribed.

A warning! Once you call CompositeSubscription.unsubscribe() the object is unusable, as it will automatically unsubscribe anything you add to it afterwards! You must create a new CompositeSubscription as a replacement if you plan on re-using this pattern later.

Solutions to both problems involve adding code; I'm hoping that someday a genius comes by and figures out how to solve these problems without all the boilerplate.