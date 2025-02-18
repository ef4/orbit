---
title: What's new in v0.17
---

This is a distillation of what's new in Orbit v0.17, intended as a reference for
developers who need to upgrade their apps and libraries from v0.16.

If you're brand new to Orbit yourself, you may wish to skip this section in
order to explore Orbit's latest features in a broader context.

## New Site + API Reference

v0.17 is Orbit's first release that comes with [API docs](./api/index.md) for
all its packages. These docs are generated by [TypeDoc](https://typedoc.org/)
from Orbit's typings and code annotations. Although a bit sparse for now, this
reference should only improve with time and help from the community.
Contributions will be most appreciated!

## Improved, strict typings throughout

The TypeScript in all of Orbit's packages has been improved to the extent that
it is now all compiled with the
[strict](https://www.typescriptlang.org/tsconfig#strict) flag. This has allowed
us to refactor more confidently, improve our documentation, and provide a
better developer experience all around.

## Extraction of `@orbit/records` from `@orbit/data`

As part of the push to improve typings, it became clear that
[`@orbit/data`](./api/data/index.md) contains a number of interfaces and classes
that could prove useful for _any_ type of data, not just records. Thus,
record-specific types and classes were extracted into a new package:
[`@orbit/records`](./api/records/index.md).

Please review the [exports](./api/records/modules.md) from `@orbit/records` for
a complete listing of classes, interfaces, and other types that have been moved
to this new package.

Be aware that several exports have been renamed to be explicit about being
record-specific. For instance, `Schema` is now `RecordSchema`, so you'll want to
make this refactor:

```diff
- import { Schema } from '@orbit/data';
+ import { RecordSchema } from '@orbit/records';
```

Apologies for this breaking change and the refactoring it requires. We're trying
to settle the scope of each package prior to v1.0.

:::caution Breaking change

Please review all your direct imports from `@orbit/data` and replace them as
needed with imports from `@orbit/records`.

:::

## Singular vs. multi-expression queries

In v0.16, each `Query` could only have a single `expression`:

```typescript
// v0.16
export interface Query {
  id: string;
  expression: QueryExpression;
  options?: any;
}
```

Now, [`Query`](./api/data/interfaces/Query.md) is typed as follows, with
`expressions` that can be singular or an array of query expressions:

```typescript
// v0.17
export interface Query<QE extends QueryExpression> {
  id: string;
  expressions: QE | QE[];
  options?: RequestOptions;
}
```

This allows sources, such as
[`JSONAPISource`](./api/jsonapi/classes/JSONAPISource.md), to optionally perform
these expressions in parallel, which it does now by default.

Now that queries can contain multiple expressions just like transforms can
contain multiple operations, there needs to be a clear and consistent way to
build them. And likewise, the expectation needs to be clear about the form
in which results should be returned.

Here's a single expression to a query builder, which can be expected to return
a single result:

```typescript
const earth = source.query((q) =>
  q.findRecord({ type: 'planet', id: 'earth' })
);
```

That same expression could be passed in an array, which will cause results to be
returned in an array:

```typescript
const [earth] = source.query((q) => [
  q.findRecord({ type: 'planet', id: 'earth' })
]);
```

And of course, that array could be expanded to include more than one expression:

```typescript
const [earth, jupiter, saturn] = source.query((q) => [
  q.findRecord({ type: 'planet', id: 'earth' }),
  q.findRecord({ type: 'planet', id: 'jupiter' }),
  q.findRecord({ type: 'planet', id: 'saturn' })
]);
```

As mentioned above, this query may be handled with 3 parallel requests, but will
only resolve when all have completed successfully.

:::caution Breaking change

Although most developers typically do not interact with queries directly, if
you do it's important to note the change from `expression` -> `expressions`.

:::

## Singular vs. multi-operation transforms

All the patterns mentioned above for queries also apply to transforms.

A single operation provided to a transform builder will return a single result:

```typescript
const earth = source.update((t) =>
  t.addRecord({ type: 'planet', id: 'earth' })
);
```

The same expression passed in an array will cause results to be returned in an
array:

```typescript
const [earth] = source.update((t) => [
  t.addRecord({ type: 'planet', id: 'earth' })
]);
```

And as before, multi-operation transforms will produce an array of results:

```typescript
const [earth, jupiter, saturn] = source.update((t) => [
  t.addRecord({ type: 'planet', id: 'earth' }),
  t.addRecord({ type: 'planet', id: 'jupiter' }),
  t.addRecord({ type: 'planet', id: 'saturn' })
]);
```

The [`Transform`](./api/data/interfaces/Transform.md) interface has changed
subtly such that `operations` can now be singular or an array, to parallel
`Query#expressions`:

```typescript
// v0.17
export interface Transform<O extends Operation> {
  id: string;
  operations: O | O[];
  options?: RequestOptions;
}
```

:::caution Breaking changes

The change that allows `Transform`'s `operations` to be singular is breaking.
You may wish to use a utility function such as
[`toArray`](./api/utils/modules.md#toarray) to interact with `operations`
uniformly as an array.

Also note that, in v0.16, calling `update` with a single operation in an array
would return a singular result. It will now return that same result as the
single member of an array.

:::

## Full vs. data-only responses

All requests (queries and updates) can now be made with a `{ fullResponse: true
}` option to receive responses as a
[`FullResponse`](./api/data/interfaces/FullResponse.md). Full responses include
the following members:

- `data` - the primary data that would be returned without the `fullResponse`
  option

- `details` - response details particular to the source. For a
  [`MemorySource`](./api/memory/classes/MemorySource.md), this will include
  applied and inverse operations. For a
  [`JSONAPISource`](./api/jsonapi/classes/JSONAPISource.md), this will include
  [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response)
  objects and documents.

- `transforms` - these are the transforms applied as a result of this request.
  They are always emitted with a `transform` event, which hooks into Orbit's
  sync flow.

- `sources` - a map of source-specific response details from downstream sources
  that were engaged in fulfilling this request.

It's now up to you just how much of this information you want at the call site.
The following requests will be handled the same internally:

```typescript
// Just the data
const planets = await source.query((q) => q.findRecords('planet'));

// All the details
const { data, details, transforms, sources } = await source.query(
  (q) => q.findRecords('planet'),
  { fullResponse: true }
);
```

## Improved response typings

Speaking of responses, it's now possible to type them using [TypeScript
generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) instead
of relying on type coercion (i.e. `response as Type`).

Standard data requests can type the response data:

```typescript
// query<RequestData>(queryOrExpressions, options, id?): Promise<RequestData>
const planets = await source.query<Planet[]>((q) => q.findRecords('planet'));
```

Full data requests can type the response data, details, and operation:

```typescript
// query<RequestData, RequestDetails, RequestOperation>(queryOrExpressions, options, id?): Promise<FullResponse<RequestData, RequestDetails, RequestOperation>>;
const { data, details, transforms, sources } = await source.query<
  Planet[],
  JSONAPIResponse[],
  RecordOperation
>((q) => q.findRecords('planet'), { fullResponse: true });
```

## Deprecation of `Pullable` and `Pushable` interfaces

Now that responses can include full processing details, everything that was
unique to the `pull` and `push` methods on source is redundant. The `Pullable`
and `Pushable` interfaces have been deprecated to focus on the more capable
`Queryable` and `Updatable` interfaces for making requests.

One common use case for `pull` / `push` was restoring from backup:

```typescript
const transform = await backup.pull((q) => q.findRecords());
await memory.push(transform);
```

This can be achieved as follows with `query` / `sync` (or `update`):

```typescript
const allRecords = await backup.query((q) => q.findRecords());
await memory.sync((t) => allRecords.map((r) => t.addRecord(r)));
```

And if you do want access to the transforms that result from a request, specify
that you want a full response:

```typescript
const { transforms } = await source.update((t) => [
    t.addRecord(type: 'planet', attributes: { name: 'Earth' }),
    t.addRecord(type: 'planet', attributes: { name: 'Jupiter' })
  ],
  { fullResponse: true }
);
```

## Transform buffers for faster cache processing

Record-cache-based sources that interact with browser storage have had
performance issues when dealing with large datasets, especially when paired with
read/write heavy processors that ensure relationship tracking and correctness. A
new paradigm has been developed, the `RecordTransformBuffer`, that acts as a
memory buffer for these operations.

For now, using this buffer is opt-in, with the `{ useBuffer: true }` option:

```typescript
await indexeddbSource.update((t) => [
    t.addRecord(type: 'planet', attributes: { name: 'Earth' }),
    t.addRecord(type: 'planet', attributes: { name: 'Jupiter' })
  ],
  { useBuffer: true }
);
```

Performance improvements are quite promising, and stability seems solid.

:::caution

The only edge cases we've found to be concerned about are related to cascading
deletes, which are triggered when record relationships are defined with
`dependent: delete`. In those cases, the cascade may not be as complete in the
buffer as in the actual cache, so we recommend avoiding transform buffers for
now.

:::

## New serializers

Concepts of serialization have, up until now, been very specific to usage by the
`JSONAPISource`, and particularly the `JSONAPISerializer` class. This class has
been deprecated and replaced with a series of composable serializers all build
upon a simple and flexible
[`Serializer`](./api/serializers/interfaces/Serializer.md) interface. This
interface, as well as some serializers for primitives (booleans, dates,
date-times, etc.) have been published in a new package,
[`@orbit/serializers`](./api/serializers/index.md).

New serializers particular to JSON:API have also been added to
[`@orbit/jsonapi`](./api/jsonapi/index.md), including:

- `JSONAPIDocumentSerializer`
- `JSONAPIResourceSerializer`
- `JSONAPIResourceIdentitySerializer`
- `JSONAPIResourceFieldSerializer`
- `JSONAPIOperationSerializer`
- `JSONAPIOperationsDocumentSerializer`

These new serializers remove some of the default behaviors present in v0.16 -
resource fields and types in documents are no longer dasherized and pluralized,
but are left "as is" in camelized form. This lines up with the new
recommendations for the JSON:API spec and creates much less work by default.

Each of these classes can be overridden to provide custom serialization
behavior. You could then provide those custom classes when creating your source:

```typescript
const source = new JSONAPISource({
  schema,
  serializerClassFor: buildSerializerClassFor({
    [JSONAPISerializers.Resource]: MyCustomResourceSerializer,
    [JSONAPISerializers.ResourceType]: MyCustomResourceTypeSerializer
  })
});
```

Alternatively, you can use the standard serializers but provide custom settings
for those serializers. For example, here are settings that match the previous
default serialization options:

```typescript
const source = new JSONAPISource({
  schema,
  serializerSettingsFor: buildSerializerSettingsFor({
    sharedSettings: {
      // Optional: Custom `pluralize` / `singularize` inflectors that know about
      // your app's unique data.
      inflectors: {
        pluralize: buildInflector(
          { person: 'people' }, // custom mappings
          (input) => `${input}s` // naive pluralizer, specified as a fallback
        ),
        singularize: buildInflector(
          { people: 'person' }, // custom mappings
          (arg) => arg.substring(0, arg.length - 1) // naive singularizer, specified as a fallback
        )
      }
    },
    // Serialization settings according to the type of serializer
    settingsByType: {
      [JSONAPISerializers.ResourceField]: {
        serializationOptions: { inflectors: ['dasherize'] }
      },
      [JSONAPISerializers.ResourceFieldParam]: {
        serializationOptions: { inflectors: ['dasherize'] }
      },
      [JSONAPISerializers.ResourceFieldPath]: {
        serializationOptions: { inflectors: ['dasherize'] }
      },
      [JSONAPISerializers.ResourceType]: {
        serializationOptions: { inflectors: ['pluralize', 'dasherize'] }
      },
      [JSONAPISerializers.ResourceTypePath]: {
        serializationOptions: { inflectors: ['pluralize', 'dasherize'] }
      }
    }
  })
});
```

## New validators

A common source of problems for Orbit developers has been using data that is
malformed or doesn't align with a schema's expectations. This can cause
confusing errors during processing by a cache or downstream source.

To address this problem, we're introducing "validators", which are shipped in a
new package [`@orbit/validators`](./api/validators/index.md) that includes some
validators for primitive types. Validators that are record-specific have also
been included in [`@orbit/records`](./api/records/index.md).

By default, each source will build its own set of validators and use them
automatically. You can instead share a common set of validators via the
`validatorFor` settings. And you can opt-out of using validators entirely by
configuring your sources with `{ autoValidate: false }`.

## Record normalizers

When building queries and transforms, some scenarios have been more tedious than
necessary: identifying records by a key instead of `id`, for instance, or using
a model class from a lib like ember-orbit to reference a record instead of its
json identity.

A new abstraction has been added to make query and transform builders more
flexible: record normalizers. Record normalizers implement the
[`RecordNormalizer`](./api/records/interfaces/RecordNormalizer.md) interface and
convert record identities and/or data into a normalized form.

The new base normalizer now allows `{ type, key, value }` to be used anywhere
that `{ type, id }` identities can be used, which significantly reduces the
annoyance of working with remote keys.

## Synchronous change tracking in memory forks

Previously, memory source forks behaved precisely like other memory sources:
every trackable update applied at the source level (and thus async). Now, the
default (but overrideable) behavior is to track changes at the cache level in
forks. Thus synchronous changes can be made to a forked cache and then merged
back into the base source.

This improves the DX for the most common use case for forks: editing form data
in isolation before merging coalesced changes back to the base. For example:

```typescript
// (sync) fork a base memory source
let fork = source.fork();

// (sync) add jupiter synchronously to the forked source's cache
fork.cache.update((t) =>
  t.addRecord({
    type: 'planet',
    id: 'jupiter',
    attributes: { name: 'Jupiter' }
  })
);

// (async) merge changes from the fork back to its base
await source.merge(fork);

// (async) jupiter should now be in the base source (as well as its cache)
let jupiter = await source.query((q) =>
  q.findRecord({ type: 'planet', id: 'jupiter' })
);
```

If you want to continue to track changes only at the source-level and have
`merge` work only with those changes, pass the following configuration setting
when you fork a source:

```typescript
let fork = source.fork({
  cacheSettings: { trackUpdateOperations: false }
});
```

This will prevent update tracking at the cache level and will signal to `merge`
that only transforms applied at the source-level should be merged.

## New memory cache capabilities

In addition to the above improvements to memory sources, v0.17 also adds the
following methods to [`MemoryCache`](./api/memory/classes/MemoryCache.md):

* `fork` - creates a new cache based on this one.
* `merge` - merges changes from a forked cache back into this cache.
* `rebase` - resets this cache's state to that of its `base` and then replays
  any update operations.

Memory cache forking / merging / rebasing is a lighter-weight way of
"branching" changes, that can ultimately be merged back into a source.
Cache-level forking can be paired with source-level forking for a lot of
flexibility and power.

## Debug mode

A new `debug` setting has been added to the
[`Orbit`](./api/core/interfaces/OrbitGlobal.md) global, that toggles between
using a more verbose, developer-friendly "debug" mode of Orbit vs. a leaner,
more performant production mode.

** Debug mode is enabled by default. ** Some standard features of debug mode
include deprecation warnings and extra debug-friendly verifications and
messaging.

To disable debug mode:

```typescript
import { Orbit } from '@orbit/core';

// disable debug mode
Orbit.debug = false;
```

:::info

For several releases in the v0.17 beta cycle, debug mode was used to control
whether validators would be created by default. This is no longer the case
&mdash; validators will now always be used within sources and caches unless
disabled using the `autoValidate: false` setting described above. This provides
more fine-grained control over validation settings throughout your app and its
sources.

:::

## Increased reliance on The Platform™

Orbit's codebase continues to evolve with the web, adopting new ES language and
web platform features as they are released. Custom utilities have been gradually
deprecated and phased out of the codebase (e.g. `isArray` -> `Array.isArray`),
new language features such as nullish coalescing and optional chaining have been
adopted, and platform features such as
[`crypto.randomUUID`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID)
have been adopted (with a fallback implementation if unavailable).

## Contributors

Many thanks to the contributors who made v0.17 possible:

- Paweł Bator ([@jembezmamy](https://github.com/jembezmamy))
- Philipp Brumm ([@brumm](https://github.com/brumm))
- Christian ([@makepanic](https://github.com/makepanic))
- Miguel Camba ([@cibernox](https://github.com/cibernox))
- Paul Chavard ([@tchak](https://github.com/tchak))
- Michiel de Vos ([@Michiel87](https://github.com/Michiel87))
- Dan Gebhardt ([@dgeb](https://github.com/dgeb))
- Brad Jones ([@bradjones1](https://github.com/bradjones1))
- Andreas Minnich ([@enspandi](https://github.com/enspandi))
- Clemens Mueller ([@pangratz](https://github.com/pangratz))
