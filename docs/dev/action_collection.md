
---
title: Action Collection
---

# Action Collection Design

The action collection tracks all actions taken by all Chef resources.  The resources can be in recipe code, as sub-resources of custom resources or
they may be built "by hand".  Since the action collection hooks the events which are fired from the `run_action` method on Chef::Resource it does
not matter how the resources were built (as long as they were correctly passed the Chef `run_context`).

This is complementary to the resource collection itself which as incomplete statement of what might happen or has happened in the run since there are
many common ways of invoking resource actions which are not captured by how the resource collection is built.  Replying the sequence of actions in
the action collection would be closer to replaying the chef-client converge than trying to re-converge the resource collection (although both of
those models are still flawed in the presence of any imperative code that controls the shape of those objects).

This extracts common duplicated code from the Data Collection and old Resource Reporter, and is designed to be used by other consumers which
need to ask questions like "in this run, what file resources had actions fired on them?", which can then be used to answer questions like
"which files is Chef managing in this directory?".

# Usage

## Action Collection Event Hook Registration

Consumers may register an event handler which hooks the `action_collection_registration` hook.  This event is fired directly before recipes are
compiled and converged (after library loading, attributes, etc).  This is just before the earliest point in time that a resource should fire an
action so represents the latest point that a consumer should make a decision about if it needs the action collection to be enabled or not.

Consumers should hook this method, where they will be passed the action collection instance, which can be saved to reuse, and then should
register themselves with the action collection:

```ruby
  def action_collection_registration(action_collection)
    @action_collection = action_collection
    action_collection.register(self)
  end
```

## Library Registration

Any cookbook library code may also register itself with the action collection.  The action collection will be registered with the `run_context` after
it is created, so registration may be accomplished easily:

```ruby
  Chef.run_context.action_collection.register(self)
```

## Searching

There is a function `filtered_collection` which returns "slices" off of the `ActionCollection` object.  The `max_nesting` argument can be used to prune
how deep into sub-resources the returned view goes (`max_nesting: 0` will return only resources in recipe context, with any hand created resources, but
no subresources).  There are also 5 different states of the action:  `up_to_date`, `skipped`, `updated`, `failed`, `unprocessed` which can be filtered
on.  All of these are true by default, so they must be disabled to remove them from the filtered collection.

The `ActionCollection` object itself implements enumerable and returns `ActionRecord` objects (see the `ActionCollection` code for the fields exposed on
`ActionRecords`).

This would return all file resources in any state in the recipe context:

```
Chef.run_context.action_collection.filtered_collection(max_nesting: 0).select { |rec| rec.new_resource.is_a?(Chef::Resource::File) }
```

As the action collection API was initially designed around the resource reporter and data collector use cases, the searching API is currently rudimentary
and could easily lift some of the searching features on the name of the resource from the resource collection, and could use a more fluent API
for composing searches.

## Action Collection Requires Registration

If one of the prior methods is not used to register for the action collection, then the action collection will disable itself and will not compile
the action collection in order to not waste the memory overhead of tracking the actions during the run.  The Data Collector takes advantage of this
since if the run start message from the Data Collector is refused by the server, then the Data Collector disables itself, and then does not register
with the Action Collection, which would disable the Action Collection.  This makes use of the delayed hooking through the `action_collection_regsitration`
so that the Data Collector never registers itself after it is disabled.

# Implementation Details

## Resource Event Lifecycle Hooks

Resources actions fire off several events in sequence:

1. `resource_action_start` - this is always fired first
2. `resource_current_state_loaded` - this is normally always second, but may be skipped in the case of a resource which throws an exception during
`load_current_resource` (which means that the `current_resource` off the ActionRecord may be nil).
3. `resource_up_to_date` / `resource_skipped` / `resource_updated` / `resource_failed` - one of these is always called which corresponds to the state of the action.
4. `resource_completed` - this is always fired last

For skipped resources, the conditional will be saved in the ActionRecord.  For failed resources the exception is saved in the ActionRecord.

## Unprocessed Resources

The unprocessed resource concept is to report on resources which are left in the resource collection after a failure.  A successful Chef run should
never leave any unprocessed resources (`action :nothing` resources are still inspected by the resource collection and are processed).  There must be
an exception thrown during the execution of the resource collection, and the unprocessed resources were never visited by the runner that executes
the resource collection.

This list will be necessarily incomplete of any unprocessed sub-resources in custom resources, since the run was aborted before those resources
executed actions and built their own sub-resource collections.
