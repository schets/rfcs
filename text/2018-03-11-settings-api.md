# Summary

Add scoped settings controls to crossbeam so users can safely adjust GC behavior

# Motivation

Crossbeam is built for situations with strict performance requirements,
and as such should provide users with the ability to control GC behavior.
Currently, there is little a user can do to adjust the behavior of Crossbeam
which makes it unuseable in certain situations (a read-only operation can
get stuck deallocating in a GC cycle for example).

Users might want to control whether GC can happen, how much garbage is
collected at a time, how aggressively a thread advances the epoch, and
others as more features are added to Crossbeam.

Since Crossbeam sits at a relatively low level in the software stack,
much use of Crossbeam and the associated datastructures might happen
through libraries, or at least will be abstracted away from business
functionality. This means that the way in which low-level library code
manipulates settings must be aware of the context it is operating in
or be extremely careful to control the types of settings it applies.

For that reason, this api proposal includes strictness controls that
can be applied to settings so high-level code can create a space of allowed
settings and library code will be free to manipulate settings as it wishes
knowing that the user will have already put controls on allowable settings

# Detailed design

The design is fairly simple: Alongside types/values representing settings,

```rust
// Setting for whether the GC will collect or not
enum Collect {
    Collect,
    NoCollect,
}
```
There will be an idea of the strength of the setting:

```rust
// Strength of the set setting
enum Strength<T> {

    // Code in deeper scopes can change the setting freely
    Lenient,
    
    // Code in deeper scopes can only make settings more strict
    // where the definition of strict depends on the type
    AsStrongAs(T),
    
    // Code in deeper scopes cannot change the setting
    Strict,
}
```

And each setting will have an idea of what values are stricter than others.
For the purpose of this RFC, stricter settings are settings that reduce the
possible behaviors of the GC - the idea is that the current settings are
the intersection of sanctioned behaviors from higher scopes.

```rust

impl StrongerThan for Collect {
    fn stronger_than(&self, val: Collect) -> bool {
        match self {
            &Collect::Collect => true,
            &Collect::NoCollect => val == Collect::Collect,
        }
    }
}

```


```rust

// Library code

fn do_some_work(val: int) {

    // This example could easily be solved by scoping the
    // collection disabling just around the latency sensitive call
    // More comple or poorly written functions may not do that properly
    let settings = ScopedGCSettings::new();
    
    // Disable GC for a latency sensitive call
    settings.with_collect(Collect::NoCollect);
    latency_sensitive_call(val);
    
    // Re-enable gc for post-call cleanup
    settings.with_collect(Collect::Collect);
    cleanup_call();
}

fn do_bulk_work(items: &Vec<int>) {
    
    let settings = ScopedGCSettings::new();
    
    
    // Without setting the strict setting, the do_some_work
    // code would reset the GC after each call despite the wishes
    // of the caller
    settings.with_collect(Collect::NoCollect,
                          Strength::Strict);
    for val : items {
        do_some_work(val);
    }
    settings.with_collect(Collect::Collect);
    bulk_cleanup();
}

fn do_low_priority_bulk_work(items: &Vec<int>) {
    
    let settings = ScopedGCSettings::new();
    
    
    // Without setting the strict setting, the do_some_work
    // code would reset the GC after each call despite the wishes
    // of the caller
    settings.with_collect(Collect::NoCollect,
                          Strength::Strict);
    for val : items {
        do_some_work(val);
    }
    settings.with_collect(Collect::Collect);
    bulk_cleanup();
}
```

# Drawbacks

A settings api that is too flexible can induce analysis-paralysis and at extremes
create 'experts-only' settings like the Java GC

A scope-based api has the issue of a user moving the raii-guard into a
different lifetime, although there may be some lifetime controls
that prevent this.

Should strict settings controls (as opposed to AsStrongAs) exist?

This design doesn't play well with any sort of stack switching and is less
useful in the face of iterator/futures style function chaining

# Alternatives

Instead of having an api with strength restrictions, we could just introduce a normal
settings api and not deal with the problem of context-sensitive settings

# Unresolved questions

The current api only allows users to specify that settings cannot be weaker than
some value. Is the ability to specify that settings cannot be stronger than
some value useless api bloat, and if not, how do we resolve the issue o
setting conflicts?

If code tries to change a setting and fails, should there be an api which
returns an error code? This would probably just be api bloat, and users who
really need to know can query the settings after trying to change them.
