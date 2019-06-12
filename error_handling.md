## whaddup ##

### Error Handling in RxJS ###

Error handling is crucial to ensuring that your application, or more precisely the services and events occuring within it, to `fail gracefully`

#### Refresher on JS try/catch ####

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


## RxJS ##

Given that we develop our client-side app primarily in Angular, we are more used to working with Observables.  How can we treat errors as a first class citizen in our Angular services/events? 

#### catchError ####
```ts
catchError(selector: (err:any, caught: Observable<T>) => O): OperatorFunction<T, T | ObservedValueOf<O>>
```
- Takes two arguments:
    1. An error (`err`)
    2. The source observable (`caught`)

- Common patterns:
    1. Handle the error locally and return an entirely new Observable
    2. Rethrow the error further downstream
    3. Return the source observable and `restart the subscription`

<b> Caution </b><br>


#### throwError ####

```ts
throwError(error: any, scheduler?: SchedulerLike): Observable<never>
```

Returns `Observable<never>` that will never emit an event to the subscriber; it will only emit an error once it is directly subscribed to.

<b>COMMON PITFALL</b>
```js
// no error will be logged to console.error

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
Above code fails to log an error because:<br>
1. `throwError` RETURNS AN OBSERVABLE CONTAINING THE ERROR
2. `map` implicitly wraps return values in an Observable

By the time `throwError` leaves the `map` function above, it is a higher-order Observable of type:<br>
<code> Observable<Observable\<never>></code>

The subscription unwraps the outer Observable, leaving the inner Observable lacking a subscription; thus this value hits our success-handler logic.

### Healthy Error-Handling ### 

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

### Retrying A Source Observable ###

So far we have looked at how we can handle an exception to allow our Subscription Error Handler to take over.

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






