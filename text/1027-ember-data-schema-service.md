---
stage: accepted
start-date: # In format 2024-05-11T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - data
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1027
project-link:
suite: 
---

# EmberData | SchemaService

## Summary

Upgrades the SchemaService API to improve DX and support new features.
Deprecates many APIs associated to the existing SchemaService.

## Motivation

The SchemaService today is primarily a way to rationalize the schema information
contained on Models. However, as we experimented with ember-m3, GraphQL, ModelFragments
and base improvements to Models we've come to realize this information is both
feature incomplete *and* more wieldy than required.

So while building SchemaRecord, we've iterated on these APIs and the
associated Schemas to gain confidence in a redesign. This is that redesign.

## Detailed design

- [The SchemaService](#the-schemaservice)
- [Resource Schemas]()
- [Field Schemas]()
- [Transformations]()
- [Derivations]()

### The SchemaService

The SchemaService defines required methods for loading and accessing
information about a resource.

```ts
interface SchemaService {
  // Retrieval APIs
  fields(resource: { type: string } | StableRecordIdentifier): Map<string, FieldSchema>;
  transformation(name: string): Transformation;
  derivation(name: string): Derivation;
  resource(name: string): ResourceSchema;

  hasTrait(resource: { type: string } | StableRecordIdentifier, trait: string): boolean;
  hasResource(type: string): boolean;

  // Loading APIs
  registerResources(schemas: ResourceSchema[]): void;
  registerResource(schema: ResourceSchema): void;
  registerTransformation(transform: Transformation): void;
  registerDerivation(name: string, derivation: Derivation): void;
}
```

To supply a SchemaService, use the `createSchemaService` hook on the store.
This is analagous to the `createCache` hook on the store and similarly it
will only be called a single time to create the SchemaService when it is
first needed.

```ts
class extends Store {
  createSchemaService() {
    return new SchemaService();
  }
}
```

When using `@ember-data/model`, this would look like:

```ts
import { buildSchema } from '@ember-data/model/hooks';

class extends Store {
  createSchemaService() {
    return buildSchema(this);
  }
}
```

When using `@warp-drive/schema-record`, this would look like:

```ts
import { SchemaService } from '@warp-drive/schema-record/schema';

class extends Store {
  createSchemaService() {
    return new SchemaService();
  }
}
```

If migrating between two implementations, you may sometimes want to create
a delegating service, for instance, the below shows a sketch of how that
might be done for `fields` and `hasResource` methods.

```ts
class MySchemaService {
  constructor(
    private newService: SchemaService,
    private oldService: SchemaService
  ) {}

  fields(resource) {
    return this.newService.hasResource(resource.type)
      ? this.newService.fields(resource)
      : this.oldService.fields(resource);
  }

  hasResource(type) {
    return this.newService.hasResource(type) || this.oldService.hasResource(type);
  }
}
```

Generally it will be best to only have one active schema service at a time, but
when necessary this approach gives control to you to determine which resources
should be delegated to which schema service.

For users migrating from `@ember-data/model` to `@warp-drive/schema-record`, a
delegating SchemaService will be provided from `@ember-data/model/migration-support`.

This service will fallback to attempting to lookup ResourceSchema from available
Models when a resource is not available from a prior call to `registerResource`.

```ts
import { DelegatingSchemaService } from '@ember-data/model/migration-support';
import { SchemaService } from '@warp-drive/schema-record/schema';

class extends Store {
  createSchemaService() {
    const schema = new SchemaService();
    return new DelegatingSchemaService(schema);
  }
}
```

#### Utilizing the SchemaService

To utilize the schema service, the created service can be accessed via the
`schema` property on either the `store` or via the same property on the 
`capabilities` passed to a Cache.

### Resource Schemas

```ts
type ResourceSchema = {
  '@id': IdentityField | null;
  /**
   * The name of the schema
   *
   * For cacheable resources, this should be the
   * primary resource type.
   *
   * For object schemas, this should be the name
   * of the object schema. object schemas should
   * follow the following guidelines for naming
   *
   * - for globally shared objects: The pattern `$field:${KlassName}` e.g. `$field:AddressObject`
   * - for resource-specific objects: The pattern `$${ResourceKlassName}:$field:${KlassName}` e.g. `$User:$field:ReusableAddress`
   * - for inline objects: The pattern `$${ResourceKlassName}.${fieldPath}:$field:anonymous` e.g. `$User.shippingAddress:$field:anonymous`
   */
  '@type': string;
  traits: string[];
  fields: FieldSchema[];
};
```

### FieldSchema

The following is the list of valid FieldSchema.

> [!IMPORTANT]  
> `IdentityField` may only ever appear once (if at all) in a ResourceSchema's FieldSchema[]

```ts
type FieldSchema =
  | GenericField
  | IdentityField
  | LocalField
  | ObjectField
  | SchemaObjectField
  | ArrayField
  | SchemaArrayField
  | DerivedField
  | ResourceField
  | CollectionField
  | LegacyAttributeField
  | LegacyBelongsToField
  | LegacyHasManyField;
```

```ts

/**
 * A generic "field" that can be used to define
 * primitive value fields.
 *
 * Replaces "attribute" for primitive value fields.
 * Can also be used to eject from deep-tracking of
 * objects or arrays.
 *
 * A major difference between "field" and "attribute"
 * is that "type" points to a legacy transform on
 * "attribute" that a serializer *might* use, while
 * "type" points to a new-style transform on "field"
 * that a record implementation *must* use.
 */
export type GenericField = {
  kind: 'field';
  name: string;
  /** the name of the transform to use, if any */
  type?: string;
  /**
   * Options to pass to the transform, if any
   *
   * Must comply to the specific transform's options
   * schema.
   */
  options?: ObjectValue;
};

/**
 * Represents a field whose value is the primary
 * key of the resource.
 *
 * This allows any field to serve as the primary
 * key while still being able to drive identity
 * needs within the system.
 *
 * This is useful for resources that use for instance
 * 'uuid', 'urn' or 'entityUrn' or 'primaryKey' as their
 * primary key field instead of 'id'.
 */
export type IdentityField = {
  kind: '@id';

  /**
   * The name of the field that serves as the
   * primary key for the resource.
   */
  name: string;
};

/**
 * Represents a field whose value is a local
 * value that is not stored in the cache, nor
 * is it sent to the server.
 *
 * Local fields can be written to, and their
 * value is both memoized and reactive (though
 * not deep-tracked).
 *
 * Because their state is not derived from the cache
 * data or the server, they represent a divorced
 * uncanonical source of state.
 *
 * For this reason Local fields should be used sparingly.
 *
 * In the future, we may choose to only allow our
 * own SchemaRecord to utilize them.
 *
 * Example use cases that drove the creation of local
 * fields are states like `isDestroying` and `isDestroyed`
 * which are specific to a record instance but not
 * stored in the cache. We wanted to be able to drive
 * these fields from schema the same as all other fields.
 *
 * Don't make us regret this decision.
 */
export type LocalField = {
  kind: '@local';
  name: string;
  /**
   * Not currently utilized, we are considering
   * allowing transforms to operate on local fields
   */
  type?: string;
  options?: { defaultValue?: PrimitiveValue };
};

/**
 * Represents a field whose value is an object
 * with keys pointing to values that are primitive
 * values.
 *
 * If values of the keys are not primitives, or
 * if the key/value pairs have well-defined shape,
 * use 'schema-object' instead.
 */
export type ObjectField = {
  kind: 'object';
  name: string;

  /**
   * The name of a transform to pass the entire object
   * through before displaying or serializing it.
   */
  type?: string;

  /**
   * Options to pass to the transform, if any
   *
   * Must comply to the specific transform's options
   * schema.
   */
  options?: ObjectValue;
};

/**
 * Represents a field whose value is an object
 * with a well-defined structure described by
 * a non-resource schema.
 *
 * If the object's structure is not well-defined,
 * use 'object' instead.
 */
export type SchemaObjectField = {
  kind: 'schema-object';
  name: string;

  /**
   * The name of the schema that describes the
   * structure of the object.
   *
   * These schemas
   */
  type: string;

  // FIXME: would we ever need options here?
  options?: ObjectValue;
};

/**
 * Represents a field whose value is an array
 * of primitive values.
 *
 * If the array's elements are not primitive
 * values, use 'schema-array' instead.
 */
export type ArrayField = {
  kind: 'array';
  name: string;

  /**
   * The name of a transform to pass each item
   * in the array through before displaying or
   * or serializing it.
   */
  type?: string;

  /**
   * Options to pass to the transform, if any
   *
   * Must comply to the specific transform's options
   * schema.
   */
  options?: ObjectValue;
};

/**
 * Represents a field whose value is an array
 * of objects with a well-defined structure
 * described by a non-resource schema.
 *
 * If the array's elements are not well-defined,
 * use 'array' instead.
 */
export type SchemaArrayField = {
  kind: 'schema-array';
  name: string;

  /**
   * The name of the schema that describes the
   * structure of the objects in the array.
   */
  type: string;

  // FIXME: would we ever need options here?
  options?: ObjectValue;
};

/**
 * Represents a field whose value is derived
 * from other fields in the schema.
 *
 * The value is read-only, and is not stored
 * in the cache, nor is it sent to the server.
 *
 * Usage of derived fields should be minimized
 * to scenarios where the derivation is known
 * to be safe. For instance, derivations that
 * required fields that are not always loaded
 * or that require access to related resources
 * that may not be loaded should be avoided.
 */
export type DerivedField = {
  kind: 'derived';
  name: string;

  /**
   * The name of the derivation to use.
   *
   * Derivations are functions that take the
   * record, options, and the name of the field
   * as arguments, and return the derived value.
   *
   * Derivations are memoized, and are only
   * recomputed when the fields they depend on
   * change.
   *
   * Derivations are not stored in the cache,
   * and are not sent to the server.
   *
   * Derivation functions must be explicitly
   * registered with the schema service.
   */
  type: string;

  /**
   * Options to pass to the derivation, if any
   *
   * Must comply to the specific derivation's
   * options schema.
   */
  options?: ObjectValue;
};

/**
 * Represents a field that is a reference to
 * another resource.
 */
export type ResourceField = {
  kind: 'resource';
  name: string;

  /**
   * The name of the resource that this field
   * refers to. In the case of a polymorphic
   * relationship, this should be the trait
   * or abstract type.
   */
  type: string;

  /**
   * Options for resources are optional. If
   * not present, all options are presumed
   * to be falsey
   */
  options?: {
    /**
     * Whether the relationship is async
     *
     * If true, it is expected that the cache
     * data for this field will contain a link
     * that can be used to fetch the related
     * resource when needed.
     */
    async?: boolean;

    /**
     * The name of the inverse field on the
     * related resource that points back to
     * this field on this resource to form a
     * bidirectional relationship.
     *
     * If null, the relationship is unidirectional.
     */
    inverse?: string | null;

    /**
     * If this field is satisfying a polymorphic
     * relationship on another resource, then this
     * should be set to the trait or abstract type
     * that this resource implements.
     */
    as?: string;

    /**
     * Whether this field is a polymorphic relationship,
     * meaning that it can point to multiple types of
     * resources so long as they implement the trait
     * or abstract type specified in `type`.
     */
    polymorphic?: boolean;
  };
};

/**
 * Represents a field that is a reference to
 * a collection of other resources, potentially
 * paginate.
 */
export type CollectionField = {
  kind: 'collection';
  name: string;

  /**
   * The name of the resource that this field
   * refers to. In the case of a polymorphic
   * relationship, this should be the trait
   * or abstract type.
   */
  type: string;

  /**
   * Options for resources are optional. If
   * not present, all options are presumed
   * to be falsey
   */
  options?: {
    /**
     * Whether the relationship is async
     *
     * If true, it is expected that the cache
     * data for this field will contain links
     * that can be used to fetch the related
     * resources when needed.
     *
     * When false, it is expected that all related
     * resources are loaded together with this resource,
     * and that the cache data for this field will
     * contain the full list of pointers.
     *
     * When true, it is expected that the relationship
     * is paginated. If the relationship is not paginated,
     * then the cache data for "page 1" would contain the
     * full list of pointers, and loading "page 1" would
     * load all related resources.
     */
    async?: boolean;

    /**
     * The name of the inverse field on the
     * related resource that points back to
     * this field on this resource to form a
     * bidirectional relationship.
     *
     * If null, the relationship is unidirectional.
     */
    inverse?: string | null;

    /**
     * If this field is satisfying a polymorphic
     * relationship on another resource, then this
     * should be set to the trait or abstract type
     * that this resource implements.
     */
    as?: string;

    /**
     * Whether this field is a polymorphic relationship,
     * meaning that it can point to multiple types of
     * resources so long as they implement the trait
     * or abstract type specified in `type`.
     */
    polymorphic?: boolean;
  };
};

/**
 * > [!CAUTION]
 * > This Field is LEGACY
 *
 * A generic "field" that can be used to define
 * primitive value fields.
 *
 * If the field points to an object or array,
 * it will not be deep-tracked.
 *
 * Transforms when defined are legacy transforms
 * that a serializer *might* use, but their usage
 * is not guaranteed.
 */
export type LegacyAttributeField = {
  kind: 'attribute';
  name: string;
  /** the name of the transform to use, if any */
  type?: string;
  /**
   * Options to pass to the transform, if any
   *
   * Must comply to the specific transform's options
   * schema.
   */
  options?: ObjectValue;
};

/**
 *  * > [!CAUTION]
 * > This Field is LEGACY
 *
 * Represents a field that is a reference to
 * another resource.
 *
 * This is the legacy version of the `ResourceField`
 * type, and is used to represent fields that were
 */
export type LegacyBelongsToField = {
  kind: 'belongsTo';
  name: string;

  /**
   * The name of the resource that this field
   * refers to. In the case of a polymorphic
   * relationship, this should be the trait
   * or abstract type.
   */
  type: string;

  /**
   * Options for belongsTo are mandatory.
   */
  options: {
    /**
     * Whether the relationship is async
     *
     * If true, it is expected that the cache
     * data for this field will contain a link
     * or a pointer that can be used to fetch
     * the related resource when needed.
     *
     * Pointers are highly discouraged.
     */
    async: boolean;

    /**
     * The name of the inverse field on the
     * related resource that points back to
     * this field on this resource to form a
     * bidirectional relationship.
     *
     * If null, the relationship is unidirectional.
     */
    inverse: string | null;

    /**
     * If this field is satisfying a polymorphic
     * relationship on another resource, then this
     * should be set to the trait or abstract type
     * that this resource implements.
     */
    as?: string;

    /**
     * Whether this field is a polymorphic relationship,
     * meaning that it can point to multiple types of
     * resources so long as they implement the trait
     * or abstract type specified in `type`.
     */
    polymorphic?: boolean;
  };
};

/**
 * > [!CAUTION]
 * > This Field is LEGACY
 */
export type LegacyHasManyField = {
  kind: 'hasMany';
  name: string;
  type: string;

  /**
   * Options for hasMany are mandatory.
   */
  options: {
    /**
     * Whether the relationship is async
     *
     * If true, it is expected that the cache
     * data for this field will contain links
     * or pointers that can be used to fetch
     * the related resources when needed.
     *
     * When false, it is expected that all related
     * resources are loaded together with this resource,
     * and that the cache data for this field will
     * contain the full list of pointers.
     *
     * hasMany relationships do not support pagination.
     */
    async: boolean;

    /**
     * The name of the inverse field on the
     * related resource that points back to
     * this field on this resource to form a
     * bidirectional relationship.
     *
     * If null, the relationship is unidirectional.
     */
    inverse: string | null;

    /**
     * If this field is satisfying a polymorphic
     * relationship on another resource, then this
     * should be set to the trait or abstract type
     * that this resource implements.
     */
    as?: string;

    /**
     * Whether this field is a polymorphic relationship,
     * meaning that it can point to multiple types of
     * resources so long as they implement the trait
     * or abstract type specified in `type`.
     */
    polymorphic?: boolean;

    /**
     * When omitted, the cache data for this field will
     * clear local state of all changes except for the
     * addition of records still in the "new" state any
     * time the remote data for this field is updated.
     *
     * When set to `false`, the cache data for this field
     * will instead intelligently commit any changes from
     * local state that are present in the remote data,
     * leaving any remaining changes in local state still.
     */
    resetOnRemoteUpdate?: false;
  };
};
```

### Transformations

Some fields have the ability to specify a transformation to serialize/deserialize the
data for the given field when the UI accesses the value from cache or updates the value.

Transformations are stateless objects that provide this ability to *hydrate* or *transform* a
serialized cache value into a richer form for use in your App. For instance, as an Enum or Luxon
Date.

Transformations ensure that the value in the cache is always in a raw serialized form even when
your App wants to conceptualize the state as something richer.

They are also how you can provide a default value for a field when no value is present yet in
the cache.

```ts
type Transformation<T extends Value = string, PT = unknown> = {
  serialize(value: PT, options: Record<string, unknown> | null, record: OpaqueRecordType): T;
  hydrate(value: T | undefined, options: Record<string, unknown> | null, record: OpaqueRecordType): PT;
  defaultValue?(options: Record<string, unknown> | null, identifier: StableRecordIdentifier): T;
};
```

### Derivations

Derivations are functions which injest the record their associated field is defined on,
the config for that field, and produce a value as a new field.

```ts
type Derivation<R, T> = (record: R, options: Record<string, unknown> | null, prop: string) => T;
```

For example, a user's `fullName` field is often implemented as a derivation of `firstName` and `lastName`,
the fields for which might look like this:

```ts
[
  {
    name: 'firstName',
    kind: 'field',
  },
  {
    name: 'lastName',
    kind: 'field',
  },
  {
    name: 'fullName',
    type: 'concat',
    options: { fields: ['firstName', 'lastName'], separator: ' ' },
    kind: 'derived',
  },
]
```

To support `fullName`, we would register a derivation named `concat`.

```ts
function concat(record, options, prop) {
  if (!options) throw new Error(`options is required`);
  return options.fields.map((field) => record[field]).join(options.separator ?? '');
}

store.schema.registerDerivation('concat', concat);
```

Typically derivations will represent *highly reusable computations* that apply to lots of fields and resources,
we discourage using derivations for one-off scenarios: its typically best to move those into components or
other layers of the program.

In most situations, derivations should be a calculation you want to conceptually share between the API and the UI,
such that a value derived from other state on the resource can be kept fresh in both locations.

> [!TIP]
> Derivations should rarely if ever access relationships as part of their calculation.
> Moreover, if using partial fields, you must be careful to send all fields a derivation
> needs to the client when its part of the intended request result. This is somewhere
> where EmberData's branding strategy for Resource and Request types can be used to your
> app's benefit. If requesting partial data, only add the derived field to the type supplied
> to the request when all associated fields are also present.

### Locals

There is a hidden `kind` of field referred to as `locals`. We do not currently intend to expose these
for use by consuming apps outside of our own limited usage. Most usage is related to supporting the migration
from `@ember-data/model`, though there are a few fields that are likely to stick around such as `isDestroyed`.

The reason for this hidden class of fields is that `SchemaRecord` (the upcoming replacement for `@ember-data/model`)
implements absolutely *every* feature via schema. This keeps the implementation fairly simple to reason about and
debug, as we don't juggle lots of codepaths. This meant we needed a solution for state that doesn't belong in the
cache, nor ever persists to the API.

If we find that there are compelling cases for `@local` to become a public API, we will RFC making it public
at that time.

## How we teach this

The schema format requires its own documention page as well as a guide for understanding
what each field type is and how they can be utilized and composed.

The schema service requires both developer docs and a usage guide with patterns for
loading schemas in a bundled, static json, or API-delivered pattern to show common
setups.

We will need to be clear in our language to avoid confusion, as schema (the word) is
bound to not only be overloaded, but confusing. For instance, a beginner may not
immediately grasp the distinction of a resource schema vs a field schema vs a
schema service. They may not even know what resources, fields, or service patterns are!

Visual aides and code examples that show the structure of data and how it maps to various
forms of schema or usage will be indispensable.

## Drawbacks

Without a way to elegantly compose schemas and types, or tools to generate them
from an existing source of truth, the barrier to getting started with schemas is
inherently higher than the barrier to getting started with Models.

However, it is not a requirement of this RFC to solve that problem. We're hard
at work on a tool for authoring schemas in a TypeScript based DSL that compiles
them into the json format as well as richer types than you'd get with a class or
a single TS interface alone. We expect *that* tool will be what keeps the barrier
to entry low.

However, we are also investigating creating a tool to convert OpenAPI specs into
these JSON schemas and types. OpenAPI has become a fairly widely adopted
standard with good tooling support throughout the software industry, and creating
a seamless integration with it seems like a no-brainer.

## Alternatives

There are an indefinite number of existing json schema representation formats
available, including many specifically for representing API resources.

However, we don't believe these adequately map to the characteristics we want
for performance, flexibility, and feature set in no small part because they 
generally are trying to communicate something different than what we need.

We've intentionally adopted a relatively simple interface approach that
plays well with TypeScript, is quick to iterate via switch, and arbitrarily
nests as deep as it needs to go.

## Unresolved questions

None
