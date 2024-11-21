---
title: "Angular's Resource API"
description: "An example where the new Resource API helped to simplify code."
date: 2024-11-24
author: "Pascal Chorus"
category: Angular
lang: en
---

Angular 19 introduces the new Resource API to handle asynchronous data gracefully using signals. I encountered a specific use case where this API significantly improved my code. In this post, I will describe my use case with a simple example and explain how I refactored my implementation from using observables to signals, and finally to the Resource API.


## The Example

This example originates from a real project at my company. I've created a simplified version to reflect the actual problem.

Imagine an app displaying a list of Star Wars characters. When a user clicks on a character, the app navigates to a details page showing the character's information. The details page uses the character's ID from the URL to load its data. If the ID in the URL changes, the corresponding character data should reload. Additionally, the page includes a refresh button to reload data from the API. While refreshing data is artificial for this example (since the data rarely changes), it was necessary in the original project because users could manually modify the data.


## Implementation Using Observables

Here is the original implementation before signals were available in Angular:

```typescript
@Component({
    selector: 'character-detail',
    imports: [AsyncPipe],
    template: `
    <button type="button" (click)="refresh()">Refresh</button>
    
    @let character = character$ | async;
    
    @if (character) {
      <h2>{{ character.name }}</h2>
    
      <dl>
        <dt>Height</dt>
        <dd>{{ character.height }}</dd>
        <dt>Birth year</dt>
        <dd>{{ character.birth_year }}</dd>
      </dl>
    }
  `
})
export class CharacterDetailComponent {
    private readonly route = inject(ActivatedRoute);
    private readonly http = inject(HttpClient);

    private readonly refresh$ = new Subject<void>();
    protected readonly character$ = merge(this.route.params, this.refresh$)
        .pipe(
            map(() => this.route.snapshot.params['id']),
            mergeMap(characterId => this.http.get<any>(`https://swapi.dev/api/people/${characterId}/`)),
        )

    protected refresh() {
        this.refresh$.next();
    }
}
```

In this implementation:

1. The `ActivatedRoute`'s params observable emits a new value when the `id` query parameter changes.
2. When the refresh button is clicked, the `refresh$` subject emits a value.

In both cases, the character is reloaded.

Both observables (`this.route.params` and `this.refresh$`) are merged using `merge()`. When `params` emits, the current `id` is provided as value, but when `refresh$` emits, no value is emitted. Thus, the ID is retrieved from the `ActivatedRoute`'s snapshot.

While this approach works, using the `ActivatedRoute`'s snapshot to fetch the ID feels hacky.


## Moving to Signals

When signals were introduced, I decided to refactor the code, aiming for simplicity. However, this was before Angular 19, so the Resource API wasn't available yet. Here's the refactored version:

```typescript
@Component({
    selector: 'character-detail',
    template: `
    <button type="button" (click)="refresh()">Refresh</button>

    @if (character()) {
      <h2>{{ character().name }}</h2>

      <dl>
        <dt>Height</dt>
        <dd>{{ character().height }}</dd>
        <dt>Birth year</dt>
        <dd>{{ character().birth_year }}</dd>
      </dl>
    }
  `
})
export class CharacterDetailComponent {
    private readonly route = inject(ActivatedRoute);
    private readonly http = inject(HttpClient);

    private readonly refresh$ = new Subject<void>();

    private readonly routeParams = toSignal(this.route.params);
    private readonly characterId = computed(() => this.routeParams()?.['id']);
    protected readonly character = toSignal(
        merge(toObservable(this.characterId), this.refresh$)
            .pipe(
                map(() => this.route.snapshot.params['id']),
                mergeMap(characterId => this.http.get<any>(`https://swapi.dev/api/people/${characterId}/`))
            )
    );

    protected refresh() {
        this.refresh$.next();
    }
}
```

Key Changes:

* `toSignal` converts the `ActivatedRoute`'s params observable into a signal (`routeParams`).
* `characterId` is a computed value that extracts the ID from `routeParams`.
* `character` is a signal value containing the result data fetched via the http request.

The reload button still emits a new value on the subject `refresh$` when clicked. Turning it into a signal does not make sense since it does not change an actual value, it just indicates that the event of a button click happened. 

To send the request when `characterId` changes or `refresh$` emits, they need to be merged as in the example before.
Thus, `characterId` is turned back into an observable using `toObservable()`. The rest is identical to the example before.

Finally, the whole observable chain is turned back into a signal using `toSignal()` and assigned to `character`.

Although signals simplify some parts, moving back and forth between signals and observables makes the code more complex.

## Resource API for the Win

Angular 19's Resource API solves these problems elegantly:

```typescript
@Component({
    selector: 'character-detail',
    template: `
    <button type="button" (click)="refresh()">Refresh</button>

    @let character = characterResourceRef.value();
    @if (character) {
      <h2>{{ character.name }}</h2>

      <dl>
        <dt>Height</dt>
        <dd>{{ character.height }}</dd>
        <dt>Birth year</dt>
        <dd>{{ character.birth_year }}</dd>
      </dl>
    }
  `
})
export class CharacterDetailComponent {
    private readonly route = inject(ActivatedRoute);

    private readonly routeParams = toSignal(this.route.params);
    private readonly characterId = computed(() => this.routeParams()?.['id']);
    protected readonly characterResourceRef = resource({
        request: this.characterId,
        loader: ({ request: characterId }) => fetch(`https://swapi.dev/api/people/${characterId}/`).then(response => response.json())
    });

    protected refresh() {
        this.characterResourceRef.reload();
    }
}
```

The `resource()` function integrates the `characterId` and the loading of the data seamlessly.
It's parameter takes two properties:

1. `request`
    
    A signal that invokes the loader function to execute, i.e. whenever `this.characterId` changes, the loader function is executed.
2. `loader`
    
    A function that returns a `Promise`. The `request` value goes into the function as parameter and the actual function body contains the code to load the data asynchronously. The loader simply use the native `fetch()` function to perform the http request. You could use Angular's `HttpClient` and put it into RxJS' `firstValueFrom()` to turn it into a `Promise`.

The `resource()` function returns a `ResourceRef` that provides you with multiple useful signals:

1. `value()`: The current result value.
2. `status()`: The current `ResourceStatus`, e.g. idle, error, loading etc.
3. `isLoading()`: `true` when the data is currently loading, `false` otherwise
4. `error()`: Contains the error if an error occurred.

Calling `reload()` on the `ResourceRef` executes the loader function again.
So a click on the refresh button simply calls `reload()`.


## Conclusion

The Resource API simplifies implementations requiring data fetching based on a signal while allowing easy manual refreshes. It eliminates the need for observables in many cases, resulting in cleaner, more maintainable code.

Note: The Resource API is still experimental in Angular 19.

Find all three example implementations from above on [StackBlitz](https://stackblitz.com/~/github.com/pchorus/resource-api-example?file=src/app/character-detail-signals.component.ts).
