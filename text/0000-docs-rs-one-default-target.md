- Feature Name: `docs-rs-one-default-target`
- Start Date: 2020-05-28
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/docs.rs#343](https://github.com/rust-lang/docs.rs/issues/343)

# Summary
[summary]: #summary

Currently, docs.rs builds for 5 targets by default for each crate published to crates.io.
This RFC proposes for docs.rs to by default only build one target.

# Motivation
[motivation]: #motivation

Most crates do not compile successfully when cross-compiled.
As a result, trying to cross-compile them wastes time and resources for little benefit.
By reducing the number of targets built by default, we can speed up queue time for all crates
and additionally reduce the resource cost of operating docs.rs.

The main resource cost for docs.rs is storage.
We currently store 3.6 TB of documentation on S3, and since we never delete documentation,
this number will only go up over time. Currently, we have no trouble affording this storage,
but if Rust and docs.rs are to be sustainable for many years into the future,
we need to keep an eye on how much storage we're using.

For most crates, the documentation is the same on every platform, so there is no need to build it many times.
Building 5 targets means that builds take 5 times as long to finish,
and means that docs.rs stores 5 times as much documentation, increasing our fixed costs.
If docs.rs only builds for one target,
then queue times will go down for maintainers and storage costs will down for the docs.rs team.

See [the issue for building a single target][docs.rs#343] for more of the history behind the motivation.

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

For example, [`winapi` choose to target all Windows platforms][winapi-commit] as follows:

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

If a crate published to crates.io has neither `targets` nor `default-target` configured in `[package.metadata.docs.rs]`,
it will be built for the `x86_64-unknown-linux-gnu` target.
Otherwise, if it has `default-target` set but not `targets`, it will be built for only that target.
Otherwise, it will be built [as currently documented on docs.rs][docs.rs metadata].

## Background: Why do crates fail to build when cross-compiled?

1. The most common reason by far is that they have native C dependencies.

Most crates with C dependencies have a dependency on `pkg-config` somewhere,
and `pkg-config` does not allow cross compiling by default.
See for example the build failure for `crates-index-diff 7.0.2`:

```
   Compiling openssl-sys v0.9.57
error: failed to run custom build command for `openssl-sys v0.9.57`

Caused by:
  process didn't exit successfully: `/opt/rustwide/target/debug/build/openssl-sys-3cbb8efeb074bfbf/build-script-main` (exit code: 101)
--- stdout
cargo:rustc-cfg=const_fn
cargo:rerun-if-env-changed=I686_UNKNOWN_LINUX_GNU_OPENSSL_LIB_DIR
I686_UNKNOWN_LINUX_GNU_OPENSSL_LIB_DIR unset
cargo:rerun-if-env-changed=OPENSSL_LIB_DIR
OPENSSL_LIB_DIR unset
cargo:rerun-if-env-changed=I686_UNKNOWN_LINUX_GNU_OPENSSL_INCLUDE_DIR
I686_UNKNOWN_LINUX_GNU_OPENSSL_INCLUDE_DIR unset
cargo:rerun-if-env-changed=OPENSSL_INCLUDE_DIR
OPENSSL_INCLUDE_DIR unset
cargo:rerun-if-env-changed=I686_UNKNOWN_LINUX_GNU_OPENSSL_DIR
I686_UNKNOWN_LINUX_GNU_OPENSSL_DIR unset
cargo:rerun-if-env-changed=OPENSSL_DIR
OPENSSL_DIR unset
run pkg_config fail: "Cross compilation detected. Use PKG_CONFIG_ALLOW_CROSS=1 to override"

--- stderr
thread 'main' panicked at '

Could not find directory of OpenSSL installation, and this `-sys` crate cannot
proceed without this knowledge. If OpenSSL is installed and this crate had
trouble finding it,  you can set the `OPENSSL_DIR` environment variable for the
compilation process.

Make sure you also have the development packages of openssl installed.
For example, `libssl-dev` on Ubuntu or `openssl-devel` on Fedora.

If you're in a situation where you think the directory *should* be found
automatically, please open a bug at https://github.com/sfackler/rust-openssl
and include information about your system as well as this message.

$HOST = x86_64-unknown-linux-gnu
$TARGET = i686-unknown-linux-gnu
openssl-sys = 0.9.57

', /opt/rustwide/cargo-home/registry/src/github.com-1ecc6299db9ec823/openssl-sys-0.9.57/build/find_normal.rs:155:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
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
Currently, we only have two exceptions to this limit:
`stm32h7xx-hal` and `mimxrt1062` are allowed 20 and 25 minutes respectively.

Most of the time, when the docs.rs queue gets backed up it is not due to a single crate clogging up the queue,
but instead because many crates have been released at the same time.
Some projects release over 300 crates at the same time,
and building all of them can take several hours, delaying other crates in the meantime.

By building only a single target, we can reduce this delay significantly.
Additionally, this will reduce delays overall significantly, even for projects that only publish a single crate at a time.

## Background: How much storage is a lot?

Currently, docs.rs adds roughly 200k files a day to the server.
It has 3.2 terabytes in storage, with about 2 gigabytes added per day.
There are a total of approximately 320 million files currently stored.

## Background: Other reasons storage costs are high

- Currently, docs.rs does not compress HTML documentation,
  needlessly increasing storage costs by a great deal. This will be [fixed soon][docs.rs#780].
- Currently, docs.rs stores a [separate copy of the source code for each platform][duplicate-source].
  This is not ideal and should be fixed eventually, but is not a large contributor to our fixed costs.

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
  but over time would increase our resource usage a great deal.
- We could build only a single default target, but make it something other than `x86_64-unknown-linux-gnu`.
  This requires the docs.rs host to be same as the default or proc-macros and build scripts will break
  (see [docs.rs#491] and [docs.rs#440] for details).
  Switching the host is possible but very difficult; it would be at least a multi-month project.

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
  Note that 'affect' is ambiguous here since a crate could have different docs on different platforms
  but not intend to have differences.

# Future possibilities
[future-possibilities]: #future-possibilities

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
