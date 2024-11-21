---
title: "Angular's Resource API"
description: "TODO"
date: 2024-11-17
author: "Pascal Chorus"
category: Angular
lang: en
---

Angular 19 introduces the new Resource API to handle asynchronous data gracefully using signals.
I had a specific use case where this API helped me refactor my code for the better.
In this post, I want to describe my use case and describe how I refactored from an implementation using observables to
an approach using signals and finally moving to the resource API.


## Problem

The problem that I am describing here came up in my actual company project.
I created a simple example which reflects the actual problem in a simple way.

Imagine an app that shows a list of Star Wars characters.
When the user clicks on a character, the app routes to a component showing the details of that character.
The details page uses the character's id from the URL to load the data of the specific character.
Thus, if the ID in the URL changes, the corresponding character data should be loaded.
Furthermore, the details page contains a refresh button to reload data from the API.
In this example, this feels a bit artificial since that data changes rather rarely or even never, but in the original
company project, that could happen since data could be changed manually by the user.


## Implementation using observables

Here is the original implementation when signals weren't available in Angular:

```typescript
@Component({
    selector: 'character-detail',
    imports: [AsyncPipe],
    template: `
    <h1>Character Details</h1>

    <button type="button" (click)="refresh()">Refresh</button>

    @let character = character$ | async;

    @if (character) {
      <h2>{{ character.name }}</h2>

      <dl>
        <dt>Height</dt>
        <dd>{{ character.height }}</dd>
        <dt>Mass</dt>
        <dd>{{ character.mass }}</dd>
        <dt>Birth year</dt>
        <dd>{{ character.birth_year }}</dd>
        <dt>Gender</dt>
        <dd>{{ character.gender }}</dd>
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

So a character should always be reloaded when the query parameter `id` changes or when the user clicks the reload button.

When the query parameter changes, the `ActivatedRoute`'s params observable emits a new value. To be even more specific, we should only observe the change of the `id` parameter, but in the example above, `id` is the only parameter in the url so we skip that filter to keep the example as simple as possible.

When the reload button is clicked, it simply emits a new value on the `refresh$` subject.

To load the character's data using both observables `route.params` and `refresh$`, they are merged using the `merge()` function. When `route.params` emits a new id, this could simply be used, but when refresh$ emits a new value, it does not contain the id. Thus, there is no id value passing through the observable chain that could be used. To get around this, we simply use the `id` value of the ActivatedRoute's snapshot of the parameters and put that in the http request.

Code works correctly, but still, using the ActivatedRoute's parameters snapshot to get the id feels a bit like a hack.


## Moving to signals

When signal were introduced, I thought it might be a good idea to refactor the code, because I hoped the code gets a bit easier.
Since I tried the refactoring before Angular 19 was released, the Resource API was not available.
Here is the code I wrote:

```typescript
@Component({
    selector: 'character-detail',
    template: `
    <h1>Character Details</h1>

    <button type="button" (click)="refresh()">Refresh</button>

    @if (character()) {
      <h2>{{ character().name }}</h2>

      <dl>
        <dt>Height</dt>
        <dd>{{ character().height }}</dd>
        <dt>Mass</dt>
        <dd>{{ character().mass }}</dd>
        <dt>Birth year</dt>
        <dd>{{ character().birth_year }}</dd>
        <dt>Gender</dt>
        <dd>{{ character().gender }}</dd>
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

As you can see, the basic idea is still the same. The difference is that I try to use signals for reactive values instead of observables.
The first step is to make the `ActivatedRoute`'s params observable a signal via `routeParams = toSignal(this.route.params)`. The characterId is simply a computed value extracting the id parameter from the routeParams signal using `characterId = computed(() => this.routeParams()?.['id'])`.

The reload button still emits a new value on the `refresh$` subject when clicked. Turning it into a signal does not make sense since it does not change an actual value, it just indicates that the event of a button click took place. 

Then, the actual http request needs to be executed whenever `characterId` changes or the `refresh$` observable emits.
To do so, `this.characterId` is turned back into an observable using the function `toObservable()`.
Then, the resulting observable is merged with the `refresh$` observable as in the example above, the ActivatedRoute's params snapshot provides the id and the character is loaded via the `HttpClient`.

Finally, the whole observable chain is turned back into a signal using the function `toSignal()` and assigned to `this.character`.

The character property is used in the html template to show the character's data.

Of course, this approach uses signals, but it still uses observables at its core.
Moving back and forth between signals and observables makes it even a bit worse in my opinion.

## Resource API for the win

```typescript
@Component({
    selector: 'character-detail',
    template: `
    <h1>Character Details</h1>

    <button type="button" (click)="refresh()">Refresh</button>

    @let character = characterResourceRef.value();
    @if (character) {
      <h2>{{ character.name }}</h2>

      <dl>
        <dt>Height</dt>
        <dd>{{ character.height }}</dd>
        <dt>Mass</dt>
        <dd>{{ character.mass }}</dd>
        <dt>Birth year</dt>
        <dd>{{ character.birth_year }}</dd>
        <dt>Gender</dt>
        <dd>{{ character.gender }}</dd>
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

* Request using a parameter from path parameter
  * Example: Detail page for person in Star Wars
  * ID is parameter in the URL
  * When parameter changes, data should reload
  * When reload button is clicked, data should be reloaded

* Original approach with observables
* When moving to signals it gets even more complicated
* resource() for the win