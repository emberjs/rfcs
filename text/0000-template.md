- Stage: Accepted
- Start Date: 2022-02-12
- Relevant Team(s): ember-data 
- RFC PR: https://github.com/emberjs/rfcs/pull/642
- Tracking: (leave this empty)

# Simplify Schema Definition Service methods in Ember Data

## Summary

This RFC is an amendment to the Custom Model Classes RFC (https://github.com/emberjs/rfcs/pull/487).
Based on implementation feedback, we discovered we could simplify the arguments to
`attributesDefinitionFor` and `relationshipsDefinitionFor` to drop the string argument and always
pass in an object.


## Motivation

When implementing a schema service, code ends up easier and cleaner if it does not have to deal with
both a raw string and an object.

## Detailed design

The original RFC proposed the following interface:

```typescript
interface SchemaDefinitionService {
  // Following the existing RD implementation 
  attributesDefinitionFor(identifier: RecordIdentifier | type: string): AttributesDefinition
  
  // Following the existing RD implementation
  relationshipsDefinitionFor(identifier: RecordIdentifier | type: string): RelationshipsDefinition
  doesTypeExist(type: string): boolean
}
```

We can simplify `attributesDefinitionFor` and `relationshipsDefinitionFor` methods to always accept an object.

```typescript
interface SchemaDefinitionService {
  // Following the existing RD implementation 
  attributesDefinitionFor(identifier: RecordIdentifier | { type: string }): AttributesDefinition
  
  // Following the existing RD implementation
  relationshipsDefinitionFor(identifier: RecordIdentifier | { type: string }): RelationshipsDefinition

  doesTypeExist(type: string): boolean
}
```

## How we teach this

It simplifies the types passed in, so should be easier to teach.


## Drawbacks


## Alternatives

Keeping the existing design per the original RFC.