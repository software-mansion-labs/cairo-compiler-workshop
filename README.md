# Cairo Compiler Contributor Workshop

_Marek Kaput (Software Mansion), 2024-07-22_

The goal of this workshop is to introduce attendees to the codebase of
the [Cairo](https://github.com/starkware-libs/cairo) compiler, its architecture, and the process of contributing to it.

Attendees are expected to have a great proficiency in Rust and a basic understanding of compilation theory.
The compiler codebase is not the easy one, and it extensively uses latest compiler techniques that probably haven't been
taught in your university yet.

> [!IMPORTANT]
> This workshop has not been updated since its event date.
> The information presented here may be outdated.
> Always refer to the contribution guidelines in the compiler repository for the most up-to-date information.
> Solutions have been implemented on top of the following
> commit: [`d0273bcc8`](https://github.com/starkware-libs/cairo/commit/d0273bcc88a14c39fe61d76da7860eb25857ea83).

## Workstation setup

1. Install latest Rust `stable` and `nightly` toolchains.
2. Clone the Cairo repository, `main` branch is fine.
   > [!TIP]
   > If you feel too dumb to require applicability of exercise solutions, check out this
   commit `d0273bcc88a14c39fe61d76da7860eb25857ea83`.
   > But I warn you that if you need this, it's better for you not to touch this codebase at all.
3. Run `cargo build` to warm up compilation caches, it will take a while.

## Before we start: Making your first pull request

Here are some protips from a seasoned compiler contributor:

1. **Don't be afraid to ask questions.**
   The Cairo compiler is a complex piece of software, and it's normal to feel
   overwhelmed at first.
   If you're stuck, don't hesitate to ask for help on Slack or Telegram.
2. Use [Reviewable](https://reviewable.io/) for code review **exclusively**.
   You will make everybody's life miserable when you'll comment directly on GitHub.
   Don't be afraid of its white UI, it's just a color, but there's a configuration option to enable the dark mode.
3. Split your work into very small, self-contained commits and create separate PR for each.
    1. Do not split by implementation chunks, but split by logical changes.
    2. It's good to include a change in a test suite that is affected by your change in each PR.
    3. You can use `TODO` comments or `todo!()` macro to mark stuff that you'll resolve in following PRs in the stack.
    4. You can also start with minimal tests and do separate PRs for test suite expansion later on.
4. If you have write access to the repository:
    * Familiarize with [git spr](https://github.com/ejoffe/spr).
    * Otherwise, you're out of luck and familiarize with `git rebase`.

# Chapter 1: Introduction, compiler architecture

# Exercise 1: Adding new grammar production

Thorough this workshop we will do a set exercises which will introduce you to the Cairo compiler codebase.
We will add a new syntax to the language and implement it all the way down (almost) to Sierra generation and E2E tests.
Following our wise-man's advices from the _before we start_ section, we will split our work into small exercises that
will
nicely fit the small PRs principle.

Our new syntax is: **a `caesar` expression**.
It will take a literal short string and will encrypt it using a Caesar cipher in compile-time.
Because of time constraints, we will limit ourselves to just accept literal short strings.
Const-evaluation of arbitrary expressions is not our goal, though it's a nice homework idea!

For the following code:

```cairo
fn main() -> felt252 {
    caesar('hello world')
}
```

We want to generate the following CASM:

```cairo
[ap + 0] = 129848244432096040924311399, ap++;  // 'khoor#zruog' as short string
ret;
```

--- 

**Add a new expression production to the Cairo grammar.**

The grammar is defined by a Rust file that describes all productions, and from which a set of files providing various
types (AST, some enumerations etc.) is generated.

The definition file is located at: `crates/cairo-lang-syntax-codegen/src/cairo_spec.rs`.

Make sure to regenerate files in `crates/cairo-lang-syntax` and make your code compile.
No need to add any tests at this stage.

# Chapter 2: Parser

# Exercise 2: Lexer

Now we need to teach our lexer to recognize the new syntax, because we've added a new keyword.

**Teach the Cairo lexer to recognize the `caesar` keyword.**

Do not forget to update lexer tests and make sure they pass!

# Chapter 3: Testing framework

# Exercise 3: Parser

**Teach the Cairo parser to process `caesar(...)` expressions.**

Do not forget to add partial tree tests!
Try to think of as many edge cases as possible.

# Chapter 4: Salsa

```rust
#[salsa::database(
    DefsDatabase,
    FilesDatabase,
    LoweringDatabase,
    ParserDatabase,
    SemanticDatabase,
    SierraGenDatabase,
    SyntaxDatabase
)]
pub struct RootDatabase;
```

```rust
#[salsa::query_group(SyntaxDatabase)]
pub trait SyntaxGroup: FilesGroup + Upcast<dyn FilesGroup> {
    #[salsa::interned]
    fn intern_green(&self, field: Arc<GreenNode>) -> GreenId;
    #[salsa::interned]
    fn intern_stable_ptr(&self, field: SyntaxStablePtr) -> SyntaxStablePtrId;

    /// Returns the children of the given node.
    fn get_children(&self, node: SyntaxNode) -> Arc<Vec<SyntaxNode>>;
}
```

## Homework

Watch cool videos about Salsa and try to better understand how it works.

There is currently one video available on the newest version of Salsa:

- [Salsa Architecture Walkthrough](https://www.youtube.com/watch?v=vrnNvAAoQFk),
  which covers many aspects of the redesigned architecture.

There are also two videos on the older version Salsa, but they are rather
outdated:

- [How Salsa Works](https://youtu.be/_muY4HjSqVw), which gives a high-level
  introduction to the key concepts involved and shows how to use Salsa;
- [Salsa In More Depth](https://www.youtube.com/watch?v=i_IhACacPRY), which digs
  into the incremental algorithm and explains -- at a high-level -- how Salsa is
  implemented.


# Exercise 4: Semantic model

**Implement semantic model for the `caesar(...)` expression.**

Remember, semantic model describes "meaning" of the code.
Do not encrypt the value here, just store input.

Don't forget about adding tests! Just a happy-path is fine for us here.

> [!TIP]
> You can copy quite a lot from short string semantic model here.

> [!TIP]
> You will likely stumble upon a cryptic compilation error here.
> You will have to duplicate some line in some macro.

# Chapter 5: CairoLS

```rust
/// The Cairo compiler Salsa database tailored for language server usage.
#[salsa::database(
    DefsDatabase,
    FilesDatabase,
    LoweringDatabase,
    ParserDatabase,
    SemanticDatabase,
    SyntaxDatabase,
    DocDatabase
)]
pub struct AnalysisDatabase;
```

# Exercise 5: Lowering

**Lower caesar expressions to encrypted literals in the lowering model.**

Add a test to verify your solution works!

With 3R Caesar cipher, the short string `'foobar'` (`112628796121458` as felt) encrypts to `'irredu'` (`115940266435701` as felt).

> [!TIP]
> Check out `lower_expr_literal_helper`.

Here's example Caesar cipher implementation in Rust which works on short strings represented as `BigInt`s (which is the
case in the lowering code):

```rust
fn caesar(secret: &BigInt) -> BigInt {
    let (sign, mut bytes) = secret.to_bytes_be();
    for byte in &mut bytes {
        *byte = byte.wrapping_add(3);
    }
    BigInt::from_bytes_be(sign, &bytes)
}
```

# Chapter 6: Sierra generation in a nutshell

- This is implemented in the `SierraGen` Salsa group.
- But! I have a happy message for all of you!
- I don't know how it works, and you don't need to know it either. 🥳

# Exercise 6: The grand finale: an end-to-end test

Congratulations! You've made it to the final exercise.
This one is trivial but very satisfying.

**Add an E2E test for your new syntax.**
