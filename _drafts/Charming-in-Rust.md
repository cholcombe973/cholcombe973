---
layout: post
title: Charming in Rust
---
Today all charms in the juju charm store are Python except for one. The Gluster charm I wrote is the first compiled charm to be accepted into the [charm store](https://jujucharms.com/gluster/). Don't deploy that version though because it is
quite old now.
Why write a charm in Rust? The story starts a long time ago.

I was a new employee at Canonical. I learned about Juju which is a modeling tool that enables applications to talk to one another. This becomes a powerful concept when applied to big software like OpenStack. Openstack has many components and many of them need to interact. Juju makes this easy. In learning about juju I found out that charms (a Juju application) could be written in any language. I thought this was an amazing idea. I had recently worked at Facebook where thrift was used. This allowed teams to write in nearly any language they wanted. Being the storage team lead I looked at the state of the Juju storage charms. Ceph, Gluster, Swift and others. Ceph and Swift were written in Python. Gluster at the time was written in Bash. The charm needed work and I knew that fixing it up would enable a whole new market to open up.

Greenfield programming can be really exciting. So many possibilities! I started writing the new charm in Python. Gluster is unfortunate to program against because there's either the undocumented XML mode or CLI screen scraping. No Rest API here! I began writing classes to model the major concepts in Gluster. I then stopped myself and did some thinking. I had previously worked with a Gluster management service written in Python. While it was powerful it also had a major flaw. This critical software had bugs that lurked but were difficult to track down. The code had been through code review, unit testing, integration testing and finally canary testing. If you're not familiar with using canaries it involves picking a few machines and upgrading them to the new code. They're observed for awhile and then the patch is deemed safe. The problem with all this is that some code paths might only be exercised very rarely. When that happens bugs can appear. Humans make bad compilers. It's as simple as that.
When I review code I'm looking for correct logic not necessarily correct type usage throughout the program. That's a boring job that was solved a long time ago with the advent of [type safety](https://en.wikipedia.org/wiki/Type_safety). After years of fighting these bugs the team I was on started gravitated towards a rewrite of the Gluster management service into a strongly typed compiled language. All this experience started to come surging back into my thoughts as I wrote this new Gluster charm. Did I want to commit myself to the same headaches as before? Collectively the Juju ecosystem team had decided that we were a Python shop. My thinking was dangerous because it would go against precedent.

After giving Go a try and playing with Rust I started to think about my experience at Nebula. I built with the help of Shevek a Ceph deployment manager in Java. I must say it was some of the best code I've created. Java while significantly different than years ago was still a non starter with many infrastructure folks. The upside would've been maximum compatibility. Yes I'm aware of the joke: compile once, debug everywhere. I decided to give Rust a real shot. It had a lot going for it. A budding ecosystem of libraries. A thriving IRC community. A powerful compiler with easy to understand error messages. Lack of garbage collection. All dependencies are built into the binary. Speed and memory safety were just bonuses. It also inferred most of the types so the code was compact. Lastly Rust makes the right way the path of least resistance. For instance the compiler will warn and can be set to error if a returned Result is not checked. Many other languages will let you simply ignore that. Think of all the bugs that have surfaced in C from not checking return codes.

Juju actually made writing a charm in Rust easy. All the interaction with juju happens through bash commands. With a few helper libraries I wrote with a coworker it became easy. The Rust  compiler is very strict and it worked to my advantage. Rather than create more burden on my team to review large chunks of code I had the compiler work out many types of errors. Their job then became to review the logic. To evaluate the approach and maintainability. In short to do what humans are good at! The other upside was I could accept patches to my charm without fear. The compiler is ensuring that the new code fits with the existing without anything breaking. Sure people can commit patches with bad logic but that will always happen. With Python a patch code that gets called by a far flung function could now break because of a type mismatch. This happens often enough in the [Ceph charm](https://jujucharms.com/ceph-mon/) that it's rather annoying at best and terrible at worst. Shipping code that may blow up in production because of a type mismatch keeps me up at night. I think in the beginning Python feels very fast. No types getting in the way. Magical type transformation where needed. No waiting on a compiler. Runs on nearly any architecture these days. After shipping code I'm constantly finding myself endlessly writing tests to check all the possible code paths. An extremely slow fuzzer if you will. For instance a bug recently surfaced with some upgrade code that I wrote that had been looked at by many people and run in production for months. Eventually it was discerned that I had a [type mismatch](https://bugs.launchpad.net/charms.ceph/+bug/1664435) that my unit tests were not surfacing.  By using a strongly typed statically compiled language instead I was building on top of a solid base.

## Code

The rust charm operates exactly the same as a Python charm. In the main function there is a match against the name the binary was called with. This then calls a function to fulfill the request:
```rust
fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() > 0 {
        // Register our hooks with the Juju library
        // If our program was called as brick-storage-attached it calls
        // the brick_attached function and runs it
        let hook_registry: Vec<juju::Hook> =
            vec![hook!("brick-storage-attached", brick_attached),
                 hook!("brick-storage-detaching", brick_detached),
                 hook!("collect-metrics", collect_metrics),
                 hook!("config-changed", config_changed),
                 hook!("create-volume-quota", enable_volume_quota),
                 hook!("delete-volume-quota", disable_volume_quota),
                 hook!("fuse-relation-joined", fuse_relation_joined),
                 hook!("list-volume-quotas", list_volume_quotas),
                 hook!("nfs-relation-joined", nfs_relation_joined),
                 hook!("server-relation-changed", server_changed),
                 hook!("server-relation-departed", server_removed),
                 hook!("set-volume-options", set_volume_options),
                 hook!("update-status", update_status)];

        // Notice no type is set here.  The compiler infers it
        let result = juju::process_hooks(hook_registry);

        if result.is_err() {
            log!(format!("Hook failed with error: {:?}", result.err()), Error);
        }
        update_status();
    }
}
```
Logging and juju status updates are handled by a macro. This was done for ergonomic reasons. Calling the macro involves much less typing and looks cleaner:
```rust
// Sends this message to the juju debug-log output
log!("Getting current_version");

// Sets the message on the console
// when `juju status` is run.
status_set!(Active "Starting volume succeeded.");
```

## Helper libraries
The gluster charm makes use of several helper libraries. Some of which I wrote.
 * https://crates.io/crates/juju - to interact with Juju
 * https://crates.io/crates/gluster - to interact with Gluster
 * https://crates.io/crates/charmhelpers - for logging to Juju functionality

## Portability
Juju charms are ideally architecture agnostic. This has been one of the bigger challenges with writing Rust. x86_64, i686, powerpc are the easiest to build for. IBM z series and arm have proven difficult. I haven't solved this yet but I'm thinking of having a bash script that detects the correct architecture or building the charm into a snapcraft snap and having that do the architecture detection.

## Continuous integration
Rust has the same flow as Python. Unit test your functions as much as possible. Integration test to flush out compatibility bugs. Ensure that tests are run in each patch to keep master working.
