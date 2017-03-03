# Relais

Relay for RESTful JSON APIs.

## Scope

### In Scope

* Data Fetching - doing the least amount of API requests
* Caching - some caching interface that may be used with `localStorage`
* Easy Intergration - should integrate into a Redux/React/Reselect stack easily
* Resource Interdependency - some resources depend on other resources to be fetched.
* Soft Cache Invalidation - when a resource is stale it should be retrieved again from the API.

### Out of Scope

* making API calls other then `GET`

## API

### `createSelector`
##### `<S, P, R, B>(...pathGetters: Array<PathGetter<P>>, transform?: (args: ...Array<B>, props: P, state: S) => Selector | R) => Selector<S, P, R>`
##### `type PathGetter<P> Array<string> | (props: P) => Array<string>`
##### `type Selector <S, P, R>(state: S, props: P) => R`

Creates a selector linked to a specific ressource retrievable through an API. It retrieves all required state through the passed `pathGetters`. Those pieces of state are then passed as arguments to `transform`. `pathGetters` can either be paths to pieces of state or path getter functions returning such paths. Path getter functions are called with `state` and `props`.

### `connect`

Wraps `react-redux`'s `connect` function. It has the same API. Internaly, it creates another HoC that makes requests to the API.

## `createResourceStore`
#### `() => ResourceStore`
#### `type ResourceStore = { fetch: SuperFetch }`
#### `type SuperFetch = (url: String, options: FetchOptions) => Observable<any>`

The `ResourceStore` manages requests. It Joins requests together


### `resourceLinker`
##### `(store: ResourceStore, baseUrl: string) => <A>(...resources: Array<Resource | ResourceFetcher>, init?: A) => A`
##### `type Resource = string`
##### `type ResourceFetcher = (rStore: ResourceStore) => `

To link a piece of state to a certain URL. The URL is generated from 

`Source` may contain `{}` as placeholders for dynamic values (e.g. `/users/{}`). These placeholders are interpreted as part of the structure of the state.

### `mountCache`
#### `(key: string, store: ReduxStore, window: defaultView, storage?: Storage) => Cache`

### `reducer`

Should be attached to the store as a reducer. It handles all changes in the store triggered by the fetcher.

## Notes

### Data fetching

In most systems read API requests can be put into five categories.

(1) specific resource (e.g. the user with ID xyz)
(2) list of resources (e.g. all users in this list of IDs)
(3) unknown set of resources (e.g. every user I have access to)
(4) sub-resource (e.g. give me the profile image of this user)
(5) aggregate endpoints returning multiple different resources (NOT supported)

These problems have to be addressed by `resourceLinker`.

(1) Is pretty straight forward. Simply take a path that contains placeholders. Let's make an example. Say we have a resource `Roads`. Our API endpoints takes requests with the pattern `/country/{countryName}/road/{roadIdentifier}`. That pattern is passed to the `resourceLinker` and attach that to `ourStore.roads`. To select a road the user creates a `PathGetter`, `['roads', 'switzerland', 'A42']`. Internally the fetcher will see that `roads` is empty, but is a linked `Resource`. It will then use the remaining two path fragments to generate the URL to request. `switzerland` will replace the first `placeholder` and `A42` the second. The result of the request will be inserted in `ourStore.roads.switzerland.A42` by dispatching an action and triggering updates.

(2) Technically the most flexible and non obtrusive way to implement this would be using the event-loop. The fetcher would delay all requests by one "cycle" of the event-loop. First it would add the request to the request pool. The request pool would be a structure looking like this `{ [string]: [TimeoutId, Array<ResourcePathFragments>] }`. If the keys are concatenated path prefixes. In our example above, this would be `roads`. If the key has an entry in the pool, the resource path fragment (`['switzerland', 'A42']`) would be appended. The timeout with the current TimeoutId would be cleared and a new timeout would be created. The id of that new timeout would be put into the first position of that pool entry. The callback of the timeout then makes the actual request. The request can't be made using a simple path pattern anymore. We can't make any assumptions about the bulk endpoint, since there's no widely recognized standard for such a thing. So instead we pass another parameter to the `resourceLinker` (called a `ResourceFetcher`). It's a function that gets the `ResourceStore` and the requested path fragments. To make an example, say a view needs to display information on two roads; `/country/switzerland/road/A8` and `/country/switzerland/road/A42`. So the view selects `['roads', 'switzerland', 'A8']` and `['roads', 'switzerland', 'A42']`. Using the described request pool mechanism, the fetcher aggregates those requests. It then checks what `ResourceFetchers` it has. A resource fetcher can either return `false` or an observable. Returning false means, the parameters don't fit the resource fetcher. In our example the `ResourceFetcher` will get `[['roads', 'switzerland', 'A42'], ['roads', 'switzerland', 'A8']]` as a second parameter and can make the appropriate built request itself request using `ResourceStore.fetch`. Our example `ResourceFetcher` may request something like `/roads?roads=switzerland:A8,switzerland:A42`. The observable returned by the `ResourceFetcher` emits tuples of paths and values. Here that could be `[['roads', 'switzerland', 'A42'], {...}]` and `[['roads', 'switzerland', 'A8'], {...}]`. The relais reducer will then set the path to the value given as the second parameter. The implementation of this pooling mechanism isn't my highest proprity, since it's more of a nice to have feature.

(3) The fetcher iterates through all registered `ResourceFetcher`s. If there's only one requested resource it first checks all URL patterns. For these it checks if the number of placeholders is equal to the lenth of the remaining path fragment. So the `ResourceFetcher` for our example might look like this `/roads` and `/country/{}/road/{}`. If the fetcher resolves `['roads']`, it sees , that there the remaining path fragemnt is zero elements long. This matches `/roads`. So it requests that URL.

(4) This can easily be done by using a `ResourceFetcher` as a funcction. Let's say we want to request `/country/switzerland/road/A42/poles`. A selector for that resource would be `['roads', 'switzerland', 'A42', 'poles']`. This would match `/country/{}/road/{}`. But it would still have a path fragment left after applying `/country/{}/road{}`. So because it's not a perfect match, we'll instead first check if any of the `ResourceFetcher` functions accept the given path fragment (['switzerland', 'A42', 'poles']). That function can then request the apropriate URL.
