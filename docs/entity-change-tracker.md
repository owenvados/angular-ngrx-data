# EntityChangeTracker

Ngrx-data tracks entity changes that haven't yet been saved on the server.
It also preserves "original values" for these changes and you can revert them with _undo actions_.

Change-tracking and _undo_ are important for applications that make _optimistic saves_.

## Optimistic versus Pessimistic save

An _optimistic save_ stores a new or changed entity in the cache _before making a save request to the server_.
It also removes an entity from the store _before making a delete request to the server_.

> The `EntityActions` whose operation names end in `_OPTIMISTIC` start
> an _optimistic_ save.

Many apps are easier to build when saves are "optimistic" because
the changes are immediately available to application code that is watching collection selectors.
The app doesn't have to wait for confirmation that the entity operation succeeded on the server.

A _pessimistic save_ doesn't update the store until the server until the server confirms that the save succeeded,
which ngrx-data then turns into a "SUCCESS" action that updates the collection.
With a _pessimistic_ save, the changes won't be available in the store

This confirmation cycle can (and usually will) take significant time and the app has to account for that gap somehow.
The app could "freeze" the UX (perhaps with a modal spinner and navigation guards) until the confirmation cycle completes.
That's tricky code to write and race conditions are inevitable.
And it's difficult to hide this gap from the user and keep the user experience responsive.

This isn't a problem with optimistic saves because the changed data are immediately available in the store.

> The developer always has the option to wait for confirmation of an optimistic save.
> But the changed entity data will be in the store during that wait.

### Save errors

The downside of optimistic save is that _the save could fail_ for many reasons including lost connection,
timeout, or rejection by the server.

When the client or server rejects the save request,
the _nrgx_ `EntityEffect.persist$` dispatches an error action ending in `_ERROR`.

**The default entity reducer methods do nothing with save errors.**

There is no issue if the operation was _pessimistic_.
The collection had not been updated so there is no obvious inconsistency between the state
of the entity in the collection and on the server.

It the operation was _optimistic_, the entity in the cached collection has been added, removed, or updated.
The entity and the collection are no longer consistent with the state on the server.

That may be a problem for your application.
If the save fails, the entity in cache no longer accurately reflects the state of the entity on the server.
While that can happen for other reasons (e.g., a different user changed the same data),
when you get a save error, you're almost certainly out-of-sync and should be able to do something about it.

Change tracking gives the developer the option to respond to a server error
by dispatching an _undo action_ for the entity (or entities) and
thereby reverting the entity (or entities) to the last known server state.

_Undo_ is NOT automatic.
You may have other save error recovery strategies that preserve the user's
unsaved changes.
It is up to you if and when to dispatch one of the `UNDO_...` actions.

## Change Tracking

The ngrx-data tracks an entity's change-state in the collection's `changeState` property.

When change tracking is enabled (the default), the `changeState` is a _primary key to_ `changeState` _map_.

> You can disable change tracking for an individual action or the collection as a whole as
> described [below](#enable-change-tracking).

### _ChangeState_

A `changeState` map adheres to the following interface

```
export interface ChangeState<T> {
  changeType: ChangeType;
  originalValue: T | undefined;
}

export enum ChangeType {
  Unchanged, // the entity has not been changed.
  Added,     // the entity was added to the collection
  Updated,   // the entity in the collection was updated
  Deleted,   // the entity is scheduled for delete and was removed from collection.
}
```

A _ChangeState_ describes an entity that changed since its last known server value.
The `changeType` property tells you how it changed.

> `Unchanged` is an _implied_ state.
> Only changed entities are recorded in the collection's `changeState` property.
> If an entity's key is not present, assume it is `Unchanged` and has not changed since it was last
> retrieved from or successfully saved to the server.

The _original value_ is the last known value from the server.
The `changeState` object holds an entity's _original value_ for _two_ of these states: _Updated_ and _Deleted_.
For an _Unchanged_ entity, the current value is the original value so there is no need to duplicate it.
There could be no original value for an entity this is added to the collection but no yet saved.

## EntityActions and change tracking.

The collection is created with an empty `changeState` map.

### Recording a change state

Many _EntityOp_ reducer methods will record an entity's change state.
Once an entity is recorded in the `changeState`, its `changeType` and `originalValue` generally do not change.
Once "added", "deleted" or "updated", an entity stays
that way until committed or undone.

Delete (remove) is a special case with special rules.
[See below](#delete).

Here are the most important `EntityOps` that record an entity in the `changeState` map:

```
// Optimistic save operations
SAVE_ADD_ONE_OPTIMISTIC
SAVE_DELETE_ONE_OPTIMISTIC
SAVE_UPDATE_ONE_OPTIMISTIC

// Cache operations
ADD_ONE
ADD_MANY
REMOVE_ONE
REMOVE_MANY
UPDATE_ONE
UPDATE_MANY
UPSERT_ONE
UPSERT_MANY
```

### Removing an entity from the _changeState_ map.

An entity which has no entry in the `ChangeState` map is presumed to be unchanged.

The _commit_ and _undo_ operations remove entries from the `ChangeState` which means, in effect, that they are "unchanged."

The **commit** operations simply remove entities from the `changeState`.
They have no other effect on the collection.

The [**undo** operations](#undo) replace entities in the collection based on
information in the `changeState` map, reverting them their last known server-side state, and removing them from the `changeState` map.
These entities become "unchanged."

An entity ceases to be in a changed state when the server returns a new version of the entity.
Operations that put that entity in the store also remove it from the `changeState` map.

Here are the operations that remove one or more specified entities from the `changeState` map.

```
QUERY_BY_KEY_SUCCESS
QUERY_MANY_SUCCESS
SAVE_ADD_ONE_SUCCESS
SAVE_ADD_ONE_OPTIMISTIC_SUCCESS,
SAVE_DELETE_ONE_SUCCESS
SAVE_DELETE_ONE_OPTIMISTIC_SUCCESS
SAVE_UPDATE_ONE_SUCCESS
SAVE_UPDATE_ONE_OPTIMISTIC_SUCCESS
COMMIT_ONE
COMMIT_MANY
UNDO_ONE
UNDO_MANY
```

### Operations that clear the _changeState_ map.

The `EntityOps` that replace or remove every entity in the collection also reset the `changeState` to an empty object.
All entities in the collection (if any) become "unchanged".

```
ADD_ALL
QUERY_ALL_SUCCESS
REMOVE_ALL
COMMIT_ALL
UNDO_ALL
```

Two of these may surprise you.

1.  `ADD_ALL` is interpreted as a cache load from a known state.
    These entities are presumed _unchanged_.
    If you have a different intent, use `ADD_MANY`.

2.  `REMOVE_ALL` is interpreted as a cache clear with nothing to save. If you have a different intent, use _removeMany_.

You can (re)set the `changeState` to anything with `EntityOp.SET_CHANGE_STATE`.

This is a super-powerful operation that you should rarely perform.
It's most useful if you've created your own entity action and are
modifying the collection in some unique way.

<a id="undo"></a>

## _Undo_ (revert) an unsaved change

You have many options for handling an optimistic save error.
One of them is to revert the change to the entity's last known state on the server by dispatching an _undo_ action.

There are three _undo_ `EntityOps` that revert entities:
`UNDO_ONE`, `UNDO_MANY` and `UNDO_ALL`.

For `UNDO_ONE` and `UNDO_MANY`, the id(s) of the entities to revert are in the action payload.

`UNDO_ALL` reverts every entity in the `changeState` map.

Each entity is reverted as follows:

* `ADDED` - Remove from the collection and discard

* `DELETED` - Add the _original value_ of the removed entity to the collection.
  If the collection is sorted, it will be moved into place.
  If unsorted, it's added to the end of the collection.

* `UPDATED` - Update the collection with the entity's _original value_.

If you try to undo/revert an entity whose id is not in the `changeState` map, the action is silently ignored.

<a id="delete"></a>

### Deleting/removing entities

There are special change tracking rules for deleting/removing an entity from the collection

#### Added entities

When you remove or delete an "added" entity, the change tracker removes the entity from the `changeState` map because there is no server state to which such an entity could be restored.

The reducer methods that delete and remove entities should immediately remove an _added entity_ from the collection.

> The default delete and remove reducer methods remove these entities immediately.

They should not send HTTP DELETE requests to the server because these entities do not exist on the server.

> The default `EntityEffects.persist$` effect does not make HTTP DELETE requests for these entities.

#### Updated entities

An entity registered in the `changeState` map as "updated"
is reclassified as "deleted".
Its `originalValue` stays the same.
Undoing the change will restore the entity to the collection in its pre-update state.

<a id="enable-change-tracking"></a>

### Enabling and disabling change tracking

You can opt-out of change tracking for a collection by setting the collection's `enableChangeTracking` flag to `false` in its `entityMetadata`.
When `false`, ngrx-data does not track any changes for this collection
and the `EntityCollection.changeState` property remains an empty object.

You can also turnoff change tracking for a specific, cache-only action by choosing one of the
"no-tracking" `EntityOps`. They all end in "\_NO_TRACK".

```
ADD_ONE_NO_TRACK
ADD_MANY_NO_TRACK
REMOVE_ONE_NO_TRACK
REMOVE_MANY_NO_TRACK
UPDATE_ONE_NO_TRACK
UPDATE_MANY_NO_TRACK
UPSERT_ONE_NO_TRACK
UPSERT_MANY_NO_TRACK
```