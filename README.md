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