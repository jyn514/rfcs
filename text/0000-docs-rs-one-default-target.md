- Feature Name: `docs-rs-one-default-target`
- Start Date: 2020-07-12
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/docs.rs#343](https://github.com/rust-lang/docs.rs/issues/343)

# Summary
[summary]: #summary

Currently, docs.rs builds documentation for 5 targets by default for each crate published to crates.io.
This RFC proposes for docs.rs to only build `x86_64-unknown-linux-gnu` by default.

# Motivation
[motivation]: #motivation

Most crates are the same on all targets, so there's no point in building them
for every target.

Most crates do not compile successfully when cross-compiled.
As a result, trying to cross-compile them wastes time and resources for little benefit.
By reducing the number of targets built by default, we can speed up queue
time for all crates and additionally reduce the resource cost of operating
docs.rs.

<!-- TODO: make this less clunky -->
<!--Don't want to accumulate docs no one looks at
<!--
The main resource cost for docs.rs is storage.
We currently store 3.6 TB of documentation on S3, and since we never delete documentation,
this number will only go up over time. Currently, we have no trouble affording this storage,
but if Rust and docs.rs are to be sustainable for many years into the future,
we need to keep an eye on how much storage we're using.
-->

For most crates, the documentation is the same on every platform, so there is no need to build it many times.
Building 5 targets means that builds take 5 times as long to finish,
and means that docs.rs stores 5 times as much documentation, increasing our fixed costs.
If docs.rs only builds for one target,
then
- queue times will go down for crate maintainers
- storage costs will down for the docs.rs team, and
- there will be fewer pages built that will never be looked at.

In particular, the very largest crates built by docs.rs are for embedded systems
(usually generated using `svd2rust`). These crates can have several gigabytes
of documentation and thousands of HTML files, all of which are the same
on the current default targets.

For crates where the documentation is different, there's a simple way to opt-in
to more targets in `Cargo.toml`:

```toml
[package.metadata.docs.rs]
targets = ["target1", "target2", ...]
```

<!--See [the issue for building a single target][docs.rs#343] for more of the history behind the motivation.-->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

[Target triples][platform support] are used to determine the platform that code will run on.
Some crates have different functionality on different platforms, and use [conditional compilation].

docs.rs has an idea of a 'default target', which is the target shown on `docs.rs/:crate/:name`.
docs.rs also builds other targets, which can be accessed by visiting `docs.rs/:crate/:name/:target`,
and are linked in a 'Platform' dropdown on each documentation page.

Currently, docs.rs builds documentation for 5 [tier 1 platforms][platform support] by default:

- `x86_64-unknown-linux-gnu`
- `i686-unknown-linux-gnu`
- `x86_64-apple-darwin`
- `i686-pc-windows-msvc`
- `x86_64-pc-windows-msvc`

After this change, docs.rs will only build one platform by default: `x86_64-unknown-linux-gnu`.
This only changes the default, you can still opt-in to more or different targets if you choose
by adding `targets = [ ... ]` to the [docs.rs metadata] in `Cargo.toml`.

All existing documentation will be kept.
No previous releases will be affected, only releases made after the RFC was merged.

For most users, this will have no impact.
Some crates that target many different platforms will be affected,
especially if their features are substantially different between platforms.
However, it is easy to opt-in to the old behavior,
and they will have plenty of advance notice before the change is made.

For example, [`winapi` chose to target all Windows platforms][winapi-commit] as follows:

```toml
targets = ["aarch64-pc-windows-msvc", "i686-pc-windows-msvc", "x86_64-pc-windows-msvc"] 
```

<!--

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

-->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

1. If a crate published to crates.io has neither `targets` nor `default-target` configured in `[package.metadata.docs.rs]`,
it will be built for the `x86_64-unknown-linux-gnu` target.
2. If it has `default-target` set but not `targets`, it will be built for `default-target` but no other platform.
3. If it has `targets` set but not `default-target`, it will be treated as if the first entry in `targets` is the default target.
4. If both are set, all `targets` will be built and `default-target` shown on `/:crate/:version`. Targets are deduplicated, so specifying the same target twice has no effect.
5. If `targets` is set to an empty list,
    - If `default-target` is unset, it will be built for `x86_64-unknown-linux-gnu`.
    - Otherwise, it will be built for `default-target`.

## Examples

- This crate will be built for `x86_64-unknown-linux-gnu`.

```toml
[package.metadata.docs.rs]
all-features = true
```

- These crates will be built for `x86_64-pc-windows-msvc`.

```toml
[package.metadata.docs.rs]
default-target = "x86_64-pc-windows-msvc"
```

```toml
[package.metadata.docs.rs]
targets = ["x86_64-pc-windows-msvc"]
```

- These crates will be built for all Windows platforms.

In this case the default target will be `aarch64-pc-windows-msvc`.

```toml
targets = ["aarch64-pc-windows-msvc", "i686-pc-windows-msvc", "x86_64-pc-windows-msvc"]
```

In this case the default target will be `x86_64-pc-windows-msvc`.

```toml
default-target = "x86_64-pc-windows-msvc"
targets = ["aarch64-pc-windows-msvc", "i686-pc-windows-msvc", "x86_64-pc-windows-msvc"] 
```

- This crates will be built for `x86_64-apple-darwin`.

```toml
default-target = "x86_64-apple-darwin"
targets = []
```

## Background: How many crates will be affected?

<!-- TODO: add more server logs with stats of target views
- percentage of views (relative to crate)
- metric for non-default target visits on grafana?
-->

The answer to this question is not clear. docs.rs can measure the number of visits to
`/:crate/:version/:platform`, but this only shows how many users are looking at pages built for different targets, not whether those pages are different. It also does not show pages which are different but have no visitors viewing the documentation.

With that said, [here is a list](https://gist.github.com/jyn514/df510c022a558d4bdffc7cf5fc478c3f) of the visits to non-default platforms between
2020-06-29 and 2020-07-12. It does not include crates with fewer than 5 visits to non-default platforms.

The list was generated as follows:

<details><summary>Shell script</summary>

```sh
zgrep -h -o '"GET [^"]*" 200 ' /var/log/nginx/access.log* | grep -v winapi | cut -d / -f 2-4 | grep -v '^crate/' | grep -f ~/targets | cut -d / -f 1 | sort | uniq -c | awk '{ if ($1 >= 5) print $0 }' | sort -k 1 -h
```

Note that winapi is excluded since it explicitly gives its targets in `Cargo.toml`.

</details>

## Background: Why do crates fail to build when cross-compiled?

1. The most common reason by far is that they have native C dependencies.

Most crates with C dependencies have a dependency on `pkg-config` somewhere,
and `pkg-config` does not allow cross compiling by default.
See for example the build failure for `crates-index-diff 7.0.2`:

```
error: failed to run custom build command for `openssl-sys v0.9.57`
--- stdout
run pkg_config fail: "Cross compilation detected. Use PKG_CONFIG_ALLOW_CROSS=1 to override"
```

Here, `pkg_config` fails because docs.rs is cross compiling from `x86_64-unknown-linux-gnu` to `i686-unknown-linux-gnu`,
and the entire build fails as a result.

2. The second most common reason is that crates do feature detection in `build.rs` via the `$TARGET` variable passed by cargo,
and assume that the host and target are the same (e.g. pass options for `link.exe` to `cc`).
This is similar to the above case, but without `pkg_config` involved.

In both these cases, it is possible to fix the crate so it will cross-compile,
but it has to be done by the crate maintainers, not by the docs.rs maintainers.
The maintainers are often not aware that their crate fails to build when cross-compiled,
or are not interested in putting in the effort to make it succeed.

## Background: How long are queue times currently?

docs.rs has a default execution time limit of 15 minutes.
At time of writing, there are only two crates with exceptions to this limit:
20 and 25 minutes respectively.

Most of the time, when the docs.rs queue gets backed up it's not because of a
single crate clogging up the queue, but instead because many crates have been
released at the same time. Some projects release over 300 crates at the same
time, and building all of them can take several hours, delaying other crates
in the meantime.

By building only a single target, we can reduce this delay significantly.
Additionally, this will reduce delays overall significantly, even for projects that only publish a single crate at a time.

<!--
## Background: How much storage is a lot?

Currently, docs.rs adds roughly 200k files a day to the server.
It has 3.2 terabytes in storage, with about 2 gigabytes added per day.
There are a total of approximately 320 million files currently stored.
-->

<!--
## Background: Other reasons storage costs are high

- Currently, docs.rs does not compress HTML documentation,
  needlessly increasing storage costs by a great deal. This will be [fixed soon][docs.rs#780].
- Currently, docs.rs stores a [separate copy of the source code for each platform][duplicate-source].
  This is not ideal and should be fixed eventually, but is not a large contributor to our fixed costs.
  -->

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

    Its interaction with other features is clear.
    It is reasonably clear how the feature would be implemented.
    Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
-->

# Drawbacks
[drawbacks]: #drawbacks

- Crates that have different docs on different platforms will be missing those docs if the maintainer takes no action.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- We could keep the status quo, building all 5 targets for all crates.
  Doing so would avoid breaking the documentation for any crate,
  but keep queue times relatively high, and over time would increase our resource usage a great deal.
<!--
- We could keep building all 5 targets but implement parallel builds.
  This would decrease queue times and avoid a single crate from clogging up the queue,
  at the cost of increasing resource costs.
  However, this would not help as much when many crates are released at the same time.
  Additionally, implementing parallel builds will be difficult from an implementation perspective;
  there is a need to balance resource usage by builds against resource usage from the docs.rs web server
  (as most readers will know, Rust compiles can be very long and resource intensive).
  Parallel builds does not conflict with this RFC, so it is possible to implement both.
  -->
<!-- - We could build only a single default target, but make it something other than `x86_64-unknown-linux-gnu`.
  This requires the docs.rs host to be same as the default or proc-macros and build scripts will break
  (see [docs.rs#491] and [docs.rs#440] for details).
  Switching the host is possible but very difficult; it would be at least a multi-month project. -->

  <!-- installing things on windows is painful -->

<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
-->

# Prior art
[prior-art]: #prior-art

- [GoDoc] has docs for only one platform (by default, Linux).
- [Racket](https://docs.racket-lang.org/) has docs for only a single platform but does not say which platform ('This is an installation-specific listing').

<!--
Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.
-->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How long should docs.rs wait before changing the default?
- How many crates will this affect?
  Note that 'affect' is ambiguous here since a crate could have different
  docs on different platforms but not intend to have differences.

# Future possibilities
[future-possibilities]: #future-possibilities

- docs.rs plans to add build logs for all targets in the future (currently it
only has logs for the default target). Building for one target would save
generating many of the logs - we'd only generate them for
targets that were explicitly requested.

<!--
- docs.rs could delete documentation for non-default targets on past releases.
  This needs some care, as the default target was only configurable
  starting in [October 2018][docs.rs#255].
  Releases prior to that are marked as having a default of `x86_64-unknown-linux-gnu`
  whether that was the intent or not.
- docs.rs could delete documentation for non-default targets on past releases,
  excluding the latest release. This gives almost all of the storage cost savings,
  while not affecting users very much since most use the latest version.
  Additionally, it could avoid deleting targets that were explicitly requested
  by `targets = ...` in Cargo.toml. This needs some care, as additional targets
  were only configurable starting in [March 2020][docs.rs#632], and it is not possible
  before that to distinguish explicitly requested targets from automatically built targets.
- docs.rs could delete documentation for non-default targets on releases older than a certain date (say a year)
- docs.rs could delete documentation for yanked releases.
-->

<!--
Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
-->

[platform support]: https://forge.rust-lang.org/release/platform-support.html
[docs.rs metadata]: https://docs.rs/about#metadata
[winapi-commit]: https://github.com/retep998/winapi-rs/commit/0090d411766ff22a2f280fc42e6a61e04780cdd4
[GoDoc]: https://godoc.org/-/about
[docs.rs#440]: https://github.com/rust-lang/docs.rs/issues/440
[docs.rs#491]: https://github.com/rust-lang/docs.rs/pull/491
[docs.rs#780]: https://github.com/rust-lang/docs.rs/pull/780
[duplicate-source]: https://github.com/rust-lang/rust/issues/67804#issuecomment-570320452
[docs.rs#343]: https://github.com/rust-lang/docs.rs/issues/343
[conditional compilation]:https://doc.rust-lang.org/reference/conditional-compilation.html
[docs.rs#255]: https://github.com/rust-lang/docs.rs/pull/255
[docs.rs#632]: https://github.com/rust-lang/docs.rs/pull/632
