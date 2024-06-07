---
stage: ready-for-release
start-date: 2024-05-11T00:00:00.000Z
release-date:
release-versions:
teams:
  - data
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1027'
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
- [Resource Schemas](#resource-schemas)
- [Field Schemas](#fieldschema)
- [Transformations](#transformations)
- [Derivations](#derivations)
- [Hashing Functions](#hashing-functions)
- [Locals](#locals)
- [Schema Defaults](#schema-defaults)
- [ModelFragments](#modelfragments)
- [Deprecations](#deprecations)

### The SchemaService

The SchemaService defines required methods for loading and accessing
information about a resource.

```ts
interface SchemaService {
  // Retrieval APIs
  fields(resource: { type: string } | StableRecordIdentifier): Map<string, FieldSchema>;
  resource(resource: { type: string } | StableRecordIdentifier): ResourceSchema;
  transformation(field: GenericField | ObjectField | ArrayField | { type: string }): Transformation;
  derivation(field: DerivedField | { type: string }): Derivation;
  hashFn(field: HashField | { type: string }): HashFn;

  hasTrait(name: string): boolean;
  resourceHasTrait(resource: { type: string } | StableRecordIdentifier, trait: string): boolean;
  hasResource(type: string): boolean;

  // Loading APIs
  registerResources(schemas: ResourceSchema[]): void;
  registerResource(schema: ResourceSchema): void;
  registerTransformation(transform: Transformation): void;
  registerDerivation(derivation: Derivation): void;
  registerHashFn(hashFn: HashFn): void;
}
```

To supply a SchemaService, use the `createSchemaService` hook on the store.
This is analagous to the `createCache` hook on the store and similarly it
will only be called a single time to create the SchemaService when it is
first needed.

> [!TIP] 
> Curious *why* `createCache` and `createSchemaService` are hooks but `lifetimes`
> and `requestManager` are properties you assign?
> Apps can have multiple stores. `lifetimes` and `requestManager` are designed to
> be shared across store instances, whereas caches (and schema) are specific to
> a store instance.

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

  hasResource({ type }) {
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
  /**
   * For primary resources, this should be an IdentityField
   * 
   * for schema-objects, this should be either a HashField or null
   */
  identity: IdentityField | HashField | null;
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
  type: string;
  traits?: string[];
  fields: FieldSchema[];
};
```

### FieldSchema

The following is the list of valid FieldSchema.

> [!IMPORTANT]  
> `IdentityField` and `HashField` may never appear in a ResourceSchema's `fields`
> these field types are *only* valid for `identity`.

```ts
type FieldSchema =
  | GenericField
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
 * Represents a specialized field whose computed value
 * will be used as the primary key of a schema-object
 * for serializability and comparison purposes.
 *
 * This field functions similarly to derived fields in that
 * it is non-settable, derived state but differs in that
 * it is only able to compute off of cache state and is given
 * no access to a record instance.
 *
 * This means that if a hashing function wants to compute its value
 * taking into account transformations and derivations it must
 * perform those itself.
 *
 * A schema-array can declare its "key" value to be `@hash` if
 * a schema-object has such a field.
 *
 * Only one hash field is permittable per schema-object, and
 * it should be placed in the `ResourceSchema`'s `@id` field
 * in place of an `IdentityField`.
 */
export type HashField = {
  kind: '@hash';

  /**
   * The name of the field that serves as the
   * hash for the resource.
   *
   * Only required if access to this value by
   * the UI is desired, it can be `null` otherwise.
   */
  name: string | null;

  /**
   * The name of a function to run to compute the hash.
   * The function will only have access to the cached
   * data for the record.
   */
  type: string;

  /**
   * Any options that should be provided to the hash
   * function.
   */
  options?: ObjectValue;
};


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
 * Currently, while we document this feature here,
 * only allow our own SchemaRecord should utilize them
 * and the feature should be considered private.
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
   * The name of the ResourceSchema that describes the
   * structure of the object.
   *
   * These `identity` for a SchemaObject shoul be either
   * a `HashField` or `null`. It should never be an IdentityField.
   */
  type: string;

  options?: {
    /**
     * Whether this SchemaObject is Polymorphic.
     *
     * If the SchemaObject is polymorphic, `options.type` must also be supplied.
     *
     * @typedoc
     */
    polymorphic?: boolean;

    /**
     * If the SchemaObject is Polymorphic, the key on the raw cache data to use
     * as the "resource-type" value for the schema-object.
     *
     * Defaults to "type".
     *
     * @typedoc
     */
    type?: string;
  };
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

  /**
   * Options for configuring the behavior of the
   * SchemaArray.
   */
  options?: {
    /**
     * Configures how the SchemaArray determines whether
     * an object in the cache is the same as an object
     * previously used to instantiate one of the schema-objects
     * it contains.
     *
     * The default is `'@identity'`.
     *
     * Valid options are:
     *
     * - `'@identity'` (default) : the cached object's referential identity will be used.
     *       This may result in significant instability when resource data is updated from the API
     * - `'@index'`              : the cached object's index in the array will be used.
     *       This is only a good choice for arrays that rarely if ever change membership
     * - `'@hash'`               : will lookup the `@hash` function supplied in the ResourceSchema for
     *       The contained schema-object and use the computed result to determine and compare identity.
     * - <field-name> (string)   : the name of a field to use as the key, only GenericFields (kind `field`)
     *       Are valid field names for this purpose. The cache state without transforms applied will be
     *       used when comparing values. The field value should be unique enough to guarantee two schema-objects
     *       of the same type will not collide.
     */
    key?: '@identity' | '@index' | '@hash' | string;

    /**
      * Whether this SchemaArray is Polymorphic.
      * 
      * If the SchemaArray is polymorphic, `options.type` must also be supplied.
      */ 
    polymorphic?: boolean;

    /**
     * If the SchemaArray is Polymorphic, the key on the raw cache data to use
     * as the "resource-type" value for the schema-object.
     * 
     * Defaults to "type".
     */
    type?: string; // default '"type"'
  };
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
 * > [!CAUTION]
 * > This Field is LEGACY
 *
 * Represents a field that is a reference to
 * another resource.
 *
 * This is the legacy version of the `ResourceField`.
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
 * 
 * Represents a field that is a reference to
 * a collection of other resources.
 *
 * This is the legacy version of the `CollectionField`.
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
export type Transformation<T extends Value = string, PT = unknown> = {
  serialize(value: PT, options: ObjectValue | null, record: OpaqueRecordInstance): T;
  hydrate(value: T | undefined, options: ObjectValue | null, record: OpaqueRecordInstance): PT;
  defaultValue?(options: ObjectValue | null, identifier: StableRecordIdentifier): T;
  [Type]: string;
};
```

### Derivations

Derivations are functions which ingest the record their associated field is defined on,
the config for that field, and produce a value as a new field.

```ts
export type Derivation<R = unknown, T = unknown> = { [Type]: string } & ((
  record: R,
  options: ObjectValue | null,
  prop: string
) => T);
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
concat[Type] = 'concat';

store.schema.registerDerivation(concat);
```

Typically derivations will represent *highly reusable computations* that apply to lots of fields and resources,
we discourage using derivations for one-off scenarios: it's typically best to move those into components or
other layers of the program.

In most situations, derivations should be a calculation you want to conceptually share between the API and the UI,
such that a value derived from other state on the resource can be kept fresh in both locations.

> [!TIP]
> Derivations should rarely if ever access relationships as part of their calculation.
> Moreover, if using partial fields, you must be careful to send all fields a derivation
> needs to the client when it's part of the intended request result. This is somewhere
> where EmberData's branding strategy for Resource and Request types can be used to your
> app's benefit. If requesting partial data, only add the derived field to the type supplied
> to the request when all associated fields are also present.

### Hashing Functions

Hashing functions are functions which injest the raw cache data for a `schema-object`
and produce a string key that represents that object's identity for purposes of serializability
and comparison. The identity need only be unique-enough that two schema-objects of the same
type can be adequately distinguished.

```ts
export type HashFn<T extends object = object> = { [Type]: string } & ((
  data: T,
  options: ObjectValue | null,
  prop: string
) => string);
```

For example 

```ts
import { Type } from '@warp-drive/core-types/symbols';

function addressHash(data) {
  return data.street + data.unit + data.city + data.state + data.zip
}
addressHash[Type] = 'address';

store.schema.registerHashFn(addressHash);
```


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

### Schema Defaults

Some packages may wish to provide utilities for enhancing ResourceSchemas with specialized fields and
derivations. Both the `@warp-drive/schema-record` package and `@ember-data/model` package are examples of this.

Since *every* behavior on a SchemaRecord is driven by schema, for a behavior to exist there needs to be
a schema field for it as well *and!* default behaviors can be optional (don't want the defaults? don't include them!).

#### For defaults with SchemaRecord:

```ts
import { withDefaults, registerDerivations } from '@warp-drive/schema-record/schema';

const upgradedResourceSchema = withDefaults(resourceSchema);

//...

// ensure derivations for use with the defaults are registered
registerDerivations(store.schema);
```

`withDefaults` adds defaults for the `constructor` property as well as for `$type`
and ensures the `ResourceSchema.identity` is set to the following IdentityField:
`{ name: 'id', kind: '@id' }`

`registerDerivations` adds derivations for the `constructor` property as well as an
`@identity` derivation that can be used to expose any information from the record's
`Identifier`, the implementation of which is shown below:

```ts
export function fromIdentity(record: SchemaRecord, options: null, key: string): asserts options;
export function fromIdentity(record: SchemaRecord, options: { key: 'lid' } | { key: 'type' }, key: string): string;
export function fromIdentity(record: SchemaRecord, options: { key: 'id' }, key: string): string | null;
export function fromIdentity(record: SchemaRecord, options: { key: '^' }, key: string): StableRecordIdentifier;
export function fromIdentity(
  record: SchemaRecord,
  options: { key: 'id' | 'lid' | 'type' | '^' } | null,
  key: string
): StableRecordIdentifier | string | null {
  const identifier = record[Identifier];
  assert(`Cannot compute @identity for a record without an identifier`, identifier);
  assert(
    `Expected to receive a key to compute @identity, but got ${String(options)}`,
    options?.key && ['lid', 'id', 'type', '^'].includes(options.key)
  );

  return options.key === '^' ? identifier : identifier[options.key];
}
fromIdentity[Type] = '@identity';
```

#### For defaults with Model:

```ts
import { withDefaults, registerDerivations } from '@ember-data/model/migration-support';

const upgradedResourceSchema = withDefaults(resourceSchema);

//...

// ensure derivations for use with the defaults are registered
registerDerivations(store.schema);
```

The defaults are primarily a mechanism to support legacy behaviors of Model while transitioning
a codebase. Most Model APIs are available via this mechanism (and, if you're curious just how advanced
derivations can get, most of these are implemented as derivations ... yes, even the functions)

-  id
-  _createSnapshot
-  adapterError
-  belongsTo
-  changedAttributes
-  constructor
-  currentState
-  deleteRecord
-  destroyRecord
-  dirtyType
-  errors
-  hasDirtyAttributes
-  hasMany
-  isDeleted
-  isEmpty
-  isError
-  isLoaded
-  isLoading
-  isNew
-  isSaving
-  isValid
-  reload
-  rollbackAttributes
-  save
-  serialize
-  unloadRecord
-  isReloading
-  isDestroying
-  isDestroyed

### ModelFragments

For users of ModelFragments, the following FieldSchemas, when used together with SchemaRecord,
are seen as the direct replacement.

- `FragmentArray` => `ArrayField` or  `SchemaArrayField` (depends on content)
  - the equivalent "key" strategy for `schema-array` is `@index`, though we believe the other
      options are likely better suited for this
  - for polymorphic arrays, the type can be specified by any field on the schema-object, but that
    field should be specified in the array's config, and should be a string value that matches a
    known `schema-object` resource-type. Note, the default is `"type"` where ModelFragments
    defaults to `"$type"`.
- `Fragment` => `ObjectField` or `SchemaObjectField` (depends on content)

## Deprecations

- `store.registerSchema` is deprecated in favor of the `createSchemaService` hook
- `store.registerSchemaDefinitionService` is deprecated in favor of the `createSchemaService` hook
- `store.getSchemaDefinitionService` is deprecated in favor of `store.schema` property
- `SchemaService.doesTypeExist` is deprecated in favor of `SchemaService.hasResource`
- `SchemaService.attributesDefinitionFor` is deprecated in favor of `SchemaService.fields`
- `SchemaService.relationshipsDefinitionFor` is deprecated in favor of `SchemaService.fields`

Since these APIs are relatively low-usage (power-user APIs mostly used by EmberData itself) and the migration
path is relatively simple, we do not foresee much churn occurring. Typical deprecation messages will print
and target `6.0`, a deprecation guide will be produced.

## How we teach this

The schema format requires its own documentation page as well as a guide for understanding
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
them into the JSON format as well as richer types than you'd get with a class or
a single TS interface alone. We expect *that* tool will be what keeps the barrier
to entry low.

However, we are also investigating creating a tool to convert OpenAPI specs into
these JSON schemas and types. OpenAPI has become a fairly widely adopted
standard with good tooling support throughout the software industry, and creating
a seamless integration with it seems like a no-brainer.

## Alternatives

There are an indefinite number of existing JSON schema representation formats
available, including many specifically for representing API resources.

However, we don't believe these adequately map to the characteristics we want
for performance, flexibility, and feature set in no small part because they 
generally are trying to communicate something different than what we need.

We've intentionally adopted a relatively simple interface approach that
plays well with TypeScript, is quick to iterate via switch, and arbitrarily
nests as deep as it needs to go.

## Unresolved questions

None
