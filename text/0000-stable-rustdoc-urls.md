- Feature Name: `stable_rustdoc_urls`
- Start Date: 2020-09-18
<!-- TODO -->
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
<!-- TODO -->
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Make the URLs that rustdoc generates stable relative to the docs being generated,
not just relative to the rustdoc version.

# Motivation
[motivation]: #motivation

Rustdoc generates a separate HTML page for each [item] in a crate.
The URL for this page currently stable relative to rustdoc; in other words,
Rustdoc guarantees that updating `rustdoc` without changing the source code will not change the URL generated.
This is a 'de facto' guarantee - it's not documented, but there's been no breaking change to the format since pre-1.0.

However, Rustdoc does _not_ currently guarantee that making a semver-compatible change to your code will preserve the same URL.
This means that, for instance, making a type an `enum` instead of a `struct` will change the URL,
even if your change is in every other way semver-compatible. After this change, Rustdoc will guarantee that the URL would stay the same.

<!-->
From a user-facing perspective, very little would change:
links within Rustdoc documentation would still work, as would existing links
and copying the URL from the address bar. However, the new format has the benefit
that it works regardless of how the name ends up in the namespace:
re-exports, different kinds of types, and any other semver-compatible changes
would not affect the URL.
-->

The primary motivation for this feature is to allow linking to a semantic version
of the docs, rather than an exact version. This has several applications:

- [docs.rs] could link to `/package/0.2/path` instead of `/package/0.2.5/path`, making the documentation users see more up-to-date
- blogs could link to exact URLs without fear of the URL breaking ([#rust-lang/rust#55160 (comment)][55160-blog])
- URLs in the standard library documentation would change less often ([rust-lang/rust#55160][55160])

Note that this is a different, but related, use case than [intra-doc links].
Intra-doc links allow linking consistently in the presence of re-exports for _relative_ links.
This is intended to be used for _absolute_ links. Additionally, this would allow linking consistently
outside of Rust code.

<!--- It avoids needlessly breaking links when an 'internal' change happens ([rust-lang/rust#55160][55160])-->

[docs.rs]: https://docs.rs/
[55160]: https://github.com/rust-lang/rust/issues/55160
[55160-blog]: https://github.com/rust-lang/rust/issues/55160#issuecomment-680751534
[intra-doc links]: https://github.com/rust-lang/rfcs/blob/master/text/1946-intra-rustdoc-links.md

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Rustdoc will make the following changes to URL structure:

- Item pages will be dependent only on the namespace, not the type of the item.

	Consider the struct `std::process::Command`.
	Currently, the URL for it looks like `std/process/struct.Command.html`.
	This RFC proposes to change the URL to `std/process/Command.t.html`.
	Pages named `kind.name.html` would still be generated (to avoid breaking existing links),
	but would immediately redirect to the new URL.

- Re-exports will generate a page pointing to the canonical version of the documentation.

	Consider the following Rust code:
	
	```rust
	pub struct Foo;
	```
	
	Rustdoc currently generates a page for this at `struct.Foo.html`.
	Now, consider what happens when you move the struct to a different module and re-export it
	(which is a semver-compatible change):

	```rust
	pub mod foo { pub struct Foo; }
	pub use foo::Foo;
	```

	This generates a page at `foo/struct.Foo.html`, but _not_ at `struct.Foo.html`.
	After this change, rustdoc will generate a page at the top level which redirects
	to the version nested in the module.

[item]: https://doc.rust-lang.org/reference/items.html

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

## Item pages will be dependent only on the namespace

Rust has three namespaces. For simplicity, this will only consider items that can be at the module level,
since function locals cannot be documented.

1. The value namespace. This includes `fn`, `const`, and `static`.
2. The type namespace. This includes `mod`, `struct`, `union`, `enum`, `trait`, `type`, struct fields, and enum variants.
3. The macro namespace. This includes `macro_rules!`, attribute macros, and derive macros.

Rust does not permit there to be overlaps within a namespace,
except for overlaps caused by globbing, where there must be a clear 'primary' import.
This means that a name and namespace is [always sufficient][find-name-namespace] to identify an item.

[find-name-namespace]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.AssociatedItems.html#method.find_by_name_and_namespace

## Re-exports will generate a page pointing to the canonical version

The redirect page will go in the same place as the re-export would be if it
were inlined with `#[doc(inline)]` after this RFC.

There will _not_ be a page generated at `kind.name.html` at the level of the re-export, since it's not possible for there to be any existing links there that were not broken.

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
-->

# Drawbacks
[drawbacks]: #drawbacks

- Rust is case-sensitive, but some filesystems (especially on Windows) are not.
This could cause overlaps in names, such as the following:

```rust
struct Command; // page generated at `Command.t.html`
enum command {} // page generated at `command.t.html`
```

Note that this is an existing problem, since you could have two items with the
same kind currently:

```rust
struct Command; // page currently generated at `struct.Command.html`
struct command {} // page currently generated at `struct.command.html`
```

This proposal makes such overlaps more likely.
This is somewhat mitigated by the `non_camel_case_types` lint, which warns
if you don't use the customary capitalization for the items.

**@nemo157** has kindly conducted a survey of the docs.rs documentation and found
that there are about 700,000 items that [currently overlap]. After this change,
that would go up to about [850,000 items that overlap][overlap-after-change].

[currently overlap]: https://ipfs.io/ipfs/QmfZatebkVFdEUYQtPaAitsBbmLKAtQgaWLciSnjtLAWfv/case-conflicts.txt.zst
[overlap-after-change]: https://ipfs.io/ipfs/QmfZatebkVFdEUYQtPaAitsBbmLKAtQgaWLciSnjtLAWfv/cased-namespace-conflicts.txt.zst

<!-- Why should we *not* do this? -->

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

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

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Rustdoc could stabilize page hashes:

- Associated items for traits will use the same hash as for types, unless there is a conflict with the hash for a type.

	A change from
	```rust
	struct S;
	impl S { fn f() {} }
	```
	to
	```rust
	struct S;
	trait T { fn f(); }
	impl T for S { fn f() {} }
	```
	is semver compatible, but currently breaks the hash (it changes from `#method.f` to `#tymethod.f`).
	Rustdoc could change it to use `#method.f` when there is no conflict with other traits or inherent associated items.
	For example, the second version of the code above would use `#method.f`, but the code below would use `#tymethod.f`
	for the version in the trait:
	```rust
	struct S;
	impl S { fn f() {} }

	trait T { fn f(); }
	impl T for S { fn f() {} }
	```

	This matches Rust semantics: `S::f()` refers to the function for the type if it exists,
	and the method for a trait it implements if not.

- Associated items for traits will contain the name of the trait if there is a conflict.

	Currently, the `from` function in both of the trait implementations has the same hash:
	```rust
	enum Int {
		A(usize),
		B(isize),
	}
	impl From<usize> for Int {
		fn from(u: usize) {
			Int::A(u)
		}
	}
	impl From<isize> for Int {
		fn from(i: isize) {
			Int::B(i)
		}
	}
	```
	This means it is _impossible_ to refer to one or the other (which has [caused trouble for intra-doc links][assoc-items]).
	Rustdoc could instead include the name and generic parameters in the hash: `#method.from-usize.from` and `method.from-isize.from`.
	It is an unresolved question how this would deal with multiple traits with the same name,
	or how this would deal with types with [characters that can't go in URL hashes][hashes] (such as `()`).
	Rustdoc could possibly use percent-encoding for the second issue.

[hashes]: https://url.spec.whatwg.org/#fragment-state
<!-- TODO: open an issue for this instead of linking to the PR -->
[assoc-items]: https://github.com/rust-lang/rust/pull/74489
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
