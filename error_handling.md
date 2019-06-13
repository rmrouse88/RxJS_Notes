## Error Handling in RxJS ##

Error handling is crucial to ensuring that your application, or more precisely the services and events occuring within it, to `fail gracefully`

### <b>try/catch/finally</b>:  A refresher on <i><b>synchronous</b></i> error handling:   ###

Most will have used the <i>synchronous</i> try/catch/finally block to handle errors:

```js
function myFunction(arg){
    try{
        const val = arg/0
        return val
    } catch(e) {
        throw e
    } finally {
        console.log('All finished with the function!')
    }
}
```

The obvious downside of try/catch/finally is that it was originally designed to handle `synchronous` errors.

#### async/await and more powerful promises ####

ES6 brought with it the `async/await` syntax to make working with promises more convenient.  The idea was simple:  Replace the chained Promise async command flow with something that looked and behaved synchronous.

```js
// legacy promise-based syntax

const myPromise = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve('whaddup');
    }, 5000)
})

myPromise.then(val => {console.log(val)}) // logs 'whaddup' after a minimum of 5 seconds
```
```js
async function findUniversalSecretAsync(){
    setTimeout(() => {
        return 42;
    }, 5000)
}

async function primaryFunction(arr){
    const universalSecret = await findUniversalSecretAsync();
    console.log(universalSecret)
}
```
What's the big deal?<br>
1.  an async function implicitly returns a `pending promise` as `Promise<pending>`
2.  The implicit promise can resolve, reject, or neither.  However, if the promise does either resolve or reject, it will trigger synchronous command flow that is accomodated by the `await` keyword
3.  What happens if a promise is rejected?  `It behaves the same as a thrown error!`
4.  The more synchronous behavior of `async/await` allows integration with legacy error-handling syntax!  `Your async functions now behave like synchronous ones!`

```js
async function findUniversalSecretAsync(){
    const truth = await new Promise(resolve => {
                                                setTimeout(() => {
                                                                resolve(42);
                                                            }, 5000)
                                                })
    return truth;
}

async function primaryFunction(arr){
    try{
        const universalSecret = await findUniversalSecretAsync();
        console.log(universalSecret)
    } catch(e){
        console.log(`error encountered and that error is ${e}`);
    } finally{
        console.log('ALL DONE!');
    }    
}
```
## <b>Reactive</b> Error Handling ##

Given that we develop our client-side app primarily in Angular, we are more used to working with Observables.  How can we treat errors as a first class citizen in our Angular services/events? 

#### <i>throwError ####

```ts
throwError(error: any, scheduler?: SchedulerLike): Observable<never>
```

Returns `Observable<never>` that will never emit an event to the subscriber; it will only emit an error once it is directly subscribed to.

<b> A COMMON PITFALL: </b> 
<h4> Understanding higher-order Observables </h4>

```js
// this sample logs no error to the console

of(1,2,3,4,5).pipe(
    map(n => {
        if (n === 4) {
            return throwError(new Error (42))
            }
            return n
        }
    ).subscribe(val => console.log(val),
                err => console.error(err)),
                () => console.log('completed!')
    )
)
```
>Above code fails to log an error because:<br>
>1. `throwError` returns an observable containing the error
>2. `map` implicitly wraps return values in an Observable

By the time `throwError` leaves the `map` function above, it is a higher-order Observable of type:  
<code> Observable<Observable\<never>></code>

The subscription unwraps the outer Observable, leaving the inner Observable untouched; thus this value fires the success-handler, not the error-handler.

>> Proper handling of throwError
```js
of(1,2,3,4,5).pipe(
    mergeMap(n => { // mergeMap will subscribe to return Observables, and merge nexted values back to the main stream
        if (n === 4) {
            return throwError(new Error (42)) //returning Observable<never>
            }
            return timer(n, n) //returning Observable<number>
        }
    ).subscribe(val => console.log(val),
                err => console.error(err)),
                () => console.log('completed!')
    )
```

## throw ##

RxJS operators wrap your input function logic in try/catch blocks:  You can use a classic `throw` within your logic without using throwError().  The difference is that `throw` doesn't return an `Observable\<never>`, simply an `error`

```ts
of(1,2,3,4).pipe(
    map(n => {
        if (n === 4){
            throw new Error('damn, it\'s 4!');
        }
        return n
    }),
    ).subscribe(val => console.log(val),
                err => console.error(`original error is ${err})`)
    )
    //  output:
    //  1
    //  2
    //  3
    //  original error is: 'damn, it\s 4!'
```


#### <i>catchError ####
```ts
catchError(selector: (err:any, caught: Observable<T>) => O): OperatorFunction<T, T | ObservedValueOf<O>>
```
> Inputs:
>   1. The error (`err`)
>   2. The source observable (`caught`)

> Outputs:
>   Observable

> Common patterns:
>   1. Handle the error locally and return an entirely new Observable
>   2. Rethrow the error further downstream
>   3. Return the source observable and `restart the subscription`

### Retrying A Source Observable ###

So far we have handled exceptions by throwing an error and allowing it to enter our subscription's error handler. an error downstream, or by catching and converting errors to a less egregious value.

But we often want to remediate errors closer to the source.  Enter retry, retryWhen, and catchError.

returning the source Observable from catchError actually means it behaves like retry, without accepting a default retry number in its constructor:

```js
of(1,2,3,4,5).pipe(
                mergeMap(n => { // mergeMap will subscribe to return Observables, and merge nexted values back to the           main stream
                    if (n === 4) {
                        return throwError(new Error (42)) //returning Observable<never>
                        }
                        return timer(n, n) //returning Observable<number>
                    }
                ),
                catchError((err, source) => {
                    return source // returning source actually creates a new subscription to the source observable
                }).
                take(30)) // take will call observer.complete() once 30 values have passed through
                .subscribe(val => console.log(val),
                            err => console.error(err),
                            () => console.log('completed!')
                )
```
## map ##

Takes in a value, applies a projection function, and returns the result of the projection function as an Observable

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

>A flattening function that will manage as many concurrent subscriptions as you would like!


>This function also provides "memory" capabilities since it can remember the value of the Outer Observable that triggered the Inner Observable

```js
// simplest form of merge-map looks a lot like map, right?
mergeMap(project: (outerValue: T, index: number) => O, 
        resultSelector?: ((outerValue: T, innerValue: ObservedValueOf<O>, outerIndex: number, innerIndex: number) => R),
        concurrent? : number)

```
Looks a lot like map, but it's project function is expected to return an Observable... an `inner Observable`
 
MergeMap will subscribe to the streams created by inner Observables, and merge values produced by those streams 

```js
concat(of(1).pipe(delay(2500)), of(3).pipe(delay(1000)), of(5).pipe(delay(1000))).pipe(
    mergeMap(outer => interval(1000).pipe(
                                        map(() => 10*outer),
                                        take(3)
                                        )
    )).subscribe(val => console.log(val));
)
```
and with memory!
```js
//courtesy of https://rxjs-dev.firebaseapp.com/api/operators/mergeMap
const letters = of('a', 'b', 'c');
const result = letters.pipe(
  mergeMap(x => interval(1000).pipe(map(i => x+i))),
);
result.subscribe(x => console.log(x));
 
// Results in the following:
// a0
// b0
// c0
// a1
// b1
// c1
// continues to list a,b,c with respective ascending integers
```

## SwitchMap ##


