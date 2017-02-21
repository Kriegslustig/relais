# Relais

Relay for RESTful JSON APIs.

## Scope

### In Scope

* Data Fetching - doing the least amount of API requests
* Caching - some caching interface that may be used with `localStorage`
* Easy Intergration - should integrate into a Redux/React/Reselect stack easily
* Resource Interdependency - Some resources depend on other resources to be fetched.

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

### `resourceLinker`
##### `(baseUrl: string) => <A>(resource: Resource, init: A) => A`
##### `type Resource = string`

To link a piece of state to a certain URL (`ressource`). `resource` may contain `{}` as placeholders for dynamic values (e.g. `/users/{}`). These placeholders are interpreted as part of the structure of the state.
