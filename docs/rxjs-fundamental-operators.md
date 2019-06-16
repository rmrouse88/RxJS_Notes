## Fundamental Operators ##

Takes in a value, applies a projection function, and returns the result of the projection function as an Observable.

> The value returned from `map` is automagically wrapped in an Observable, so you don't need to explicitly wrap unless you are trying to create a higher-order Observable.

```js
map<T,R>(project: (value, index) => R, thisArg? : any) : OperatorFunction<T,R>
```

```js
of(1,2,3,4,5).pipe(
        map(n => n*2)
    ).subscribe(val => console.log(val))  // 2, 4, 6, 8 10
```

Note that our observables are subscribed to prior to the projection, and then are re-wrapped as observables 

## mergeMap ## 

>A flattening function that will manage as many concurrent subscriptions as your browser's memory will allow!

>Takes in a value from an upstream observable (`outer values`) which triggers the creation of an inner observable which is immediately subscribed to (`inner values`); mergeMap remembers the values of the `outer values` passed in allowing you to manipulate both values in a returned Observable

Looks a lot like map, but it's `project` function is expected to return an <i>Observable</i>.

Because additional inner observables are created within the mergeMap, the function accepts a `concurrent` argument which is designed to limit the number of inner observables subscribed to at any one time. 

```ts
// simplest form of merge-map looks a lot like map, right?
mergeMap(project: (outerValue: T, index: number) => O, 
        resultSelector?: ((outerValue: T, innerValue: ObservedValueOf<O>, outerIndex: number, innerIndex: number) => R),
        concurrent? : number)

```

```ts
//courtesy of https://rxjs-dev.firebaseapp.com/api/operators/mergeMap

// only a project function provided; notice we have access to the outer value in the inner Observable

const letters = of('a', 'b', 'c');
const result = letters.pipe(
  mergeMap(x => interval(1000).pipe(map(i => x+i))),
);
result.subscribe(x => console.log(x));
```

#### ResultSelector: DEPRECATED ####

MergeMap has an overload that accepts a `resultSelector` as a second argument.



## switchMap ##

Takes in a projectFunction as its argument.  The projectFunction must return an inner Observable that will be subscribed to by the SwitchMap function.

>Unlike mergeMap, switchMap will keep only one active subscription at any one time; that subscription belongs to the most recently-created projectFunction.

```ts
switchMap(project: (outerValue: any, index: number) => O,
          resultSelector? : (outerValue: any, innerValue: ObservedValueOf<O>, outerIndex: number, innerIndex: number)
```

```ts
concat(of(1), of(3).pipe(3000), of(5).pipe(3000)).pipe(
                                                    switchMap(n => {
                                                        return timer(0, 1000).pipe(
                                                                map(() => n)
                                                                )
                                                    })
                                                    ).subscribe(val => console.log(val))
```

## concat ##

>accepts a list of Observables and subscribes to each, in order, moving through subscriptions as one completes.  


## concatMap ##

>Accepts: a projectFunction that exposes both the value from a source observable <i>and</i> an index

>Returns: an inner Observable that will be subscribed to, and whose emitted value will be merged 

Behaves similar to `concat` in the sense that a single inner Observable subscription will be maintained until it either completes or errors out, before creating a new projected Observable from the next buffered outer value in the queue.

The big difference is the presence of the map function that must, necessarily, return an inner Observable that will be subscribed to.  

## SUMMARY ##

These operators each serve to <b><i>create AND flatten</i></b> inner Observables that are triggered by an incoming source value.  But below is a brief summary of what makes the use-cases for each one unique:

1) `mergeMap`

    Used when maintaining and flattening <b>multiple</b> inner subscriptions is a necessity.  Perhaps tracking write operations to a database on a nodejs server.

2) `switchMap`

    Used when you only want to maintain a <b>single</b> inner subscription centered around the most recent source value that was received.

3) `concatMap`

    Used when you want to maintain a <b>single</b> inner subscription at a time, but wish to queue up  outer values received for creation of subsequent inner subscriptions once the active subscription completes or errors out 
