# Cairo Compiler Contributor Workshop

_Marek Kaput (Software Mansion), 2024-08-07_

The goal of this workshop is to introduce attendees to the codebase of
the [Cairo](https://github.com/starkware-libs/cairo) compiler, its architecture, and the process of contributing to it.

Attendees are expected to have a great proficiency in Rust and a basic understanding of compilation theory.
The compiler codebase is not an easy one, and it extensively uses the latest compiler techniques that probably haven't
been
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
   You will make everybody's life miserable when you comment directly on GitHub.
   Don't be afraid of its white UI, it's just a color, but there's a configuration option to enable the dark mode.
3. Split your work into very small, self-contained commits and create a separate PR for each.
    1. Do not split by implementation chunks but split by logical changes.
    2. It's good to include a change in a test suite that is affected by your change in each PR.
    3. You can use `TODO` comments or `todo!()` macro to mark stuff that you'll resolve in following PRs in the stack.
    4. You can also start with minimal tests and do separate PRs for test suite expansion later on.
4. If you have write access to the repository:
    * Familiarize with [git spr](https://github.com/ejoffe/spr).
    * Otherwise, you're out of luck and familiarize with `git rebase`.
5. This codebase relies on nightly features of rustfmt, clippy and rustdoc.
    * Use scripts in the `scripts/` directory for running these tools,
    * or configure your IDE to use nightly toolchain when doing these tasks.

# Chapter 1: Compiler architecture

- The Cairo compiler is built as a library that is split into numerous purpose-focused crates.
- There's crate for filesystem ops & collecting compiler input; there's one with parser, another one with syntax
  definition and a series of crates for code analysis and generation.
- One could identify the following stages of code compilation:
    - Filesystem access
    - Lexing
    - Parsing
    - Semantic analysis—taking syntax trees and producing sets of simple objects telling what they are.
    - Lowering: taking semantic objects for functions and producing a _Control Flow Graph_.
    - Sierra generation
    - _We will talk about how these stages are coded in a chapter about Salsa._
- The root crate for the compiler is called `cairo-lang-compiler`, and it provides a struct called `RootDatabase` which
  is the heart of the compiler.
- The most important bits from compiler output can (but doesn't have to) be a `Project` structure
    - This is serializable from `cairo_project.toml` files.
    - This is implemented in `cairo-lang-project` crate.
    - Scarb and CairoLS are working differently:
        - Scarb basically takes `Scarb.toml`, does its magic and produces a `Project` struct that is used for
          initializing the `RootDatabase.
        - CairoLS is (will be, this is in progress at the time of writing this workshop) maintaining its own graph of
          crates and their interdependencies:
            - It is populated from reading any `cairo_project.toml` or `Scarb.toml` files it encounters (the latter is
              analyzed via `scarb metadata` command).
            - State of this graph is then directly fed into LS' `RootDatabase` counterpart inputs, low-level style,
              there's no `Project` being built at any time.
- There is a set of binaries present in the compiler repository that provide compiler functionality as small programs.
    - They are largely made with Starkware internal use in mind.
    - As you all know, use Scarb.
- While there are efforts to maintain backward compatibility of Cairo as a language, the Cairo compiler library API is
  highly unstable.

# Exercise 1: Adding new grammar production

Through this workshop we will do a set exercise that will introduce you to the Cairo compiler codebase.
We will add a new syntax to the language and implement it almost all the way to Sierra generation and E2E tests.
Following our wise-man's advice from the _before we start_ section, we will split our work into small exercises that
will
nicely fit the small PRs principle.

Our new syntax is: **a `caesar` expression**.
It will take a literal short string and will encrypt it using a Caesar cipher in compile-time.
Because of time constraints, we will limit ourselves to just accept literal short strings.
Constant evaluation of arbitrary expressions is not our goal, though it's a nice homework idea!

BNF:

```bnf
Expr ::= ExprCaesar | /* ... */
ExprCaesar ::= 'caesar' '(' TerminalShortString ')'
```

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
types (AST, the `SyntaxKind` enum, etc.) is generated.

The definition file is located at: `crates/cairo-lang-syntax-codegen/src/cairo_spec.rs`.

Make sure to regenerate files in `crates/cairo-lang-syntax` and make your code compile.
You can do this with the following command:

```sh
cargo run --bin generate-syntax
```

No need to add any tests at this stage.

# Chapter 2: Parser and AST

- Cairo is using a handwritten recursive descent parser.
- The parser is located in `crates/cairo-lang-parser/src/parser.rs`.
- It emits "green nodes" and parsing diagnostics (errors).
- The parser if infallible, it always succeeds to consume source code until EOF.
    - If some production is missing, it emits a `Missing` green node and error diagnostic.
    - This enables parsing and analyzing partial trees, which enables CairoLS.

## Green nodes

```rust
pub struct GreenNode {
    pub kind: SyntaxKind,
    pub details: GreenNodeDetails,
}
pub enum GreenNodeDetails {
    Token(SmolStr),
    // GreenId is kind of a "pointer" to another green node.
    // We need to talk about Salsa to fully understand what this is.
    Node { children: Vec<GreenId>, width: TextWidth },
}
```

- Green nodes are an AST representation that is not strictly typed:
- It has no accessors, it's, for example, hard to find out the function name in the function definition subtree.
- But it is trivial to traverse in various directions and to manipulate, e.g., in formatter.

## Typed syntax tree

```rust
pub trait TypedSyntaxNode {
    const OPTIONAL_KIND: Option<SyntaxKind>;
    type StablePtr: TypedStablePtr;
    type Green;

    // Builds `<missing>` green node of this type.
    fn missing(db: &dyn SyntaxGroup) -> Self::Green;

    // Conversions from/to.
    fn from_syntax_node(db: &dyn SyntaxGroup, node: SyntaxNode) -> Self;
    fn as_syntax_node(&self) -> SyntaxNode;

    // Stable pointers, see next section.
    fn stable_ptr(&self) -> Self::StablePtr;
}
```

- On top of the green (sub) tree, a typed AST can be built.
- There are accessors, but arbitrary traversal is hard.
- Great for picking precise information from the tree.

## Stable pointers

```rust
/// Stable pointer to a node in the syntax tree.
/// Has enough information to uniquely define a node in the AST, given the tree.
/// Has undefined behavior when used with the wrong tree.
/// This is not a real pointer in the low-level sense, just a representation of the path from the
/// root to the node.
/// Stable means that when the AST is changed, pointers of unchanged items tend to stay the same.
/// For example, if a function is changed, the pointer of an unrelated function in the AST should
/// remain the same, as much as possible.
#[derive(Clone, Debug, Hash, PartialEq, Eq)]
pub enum SyntaxStablePtr {
    /// The root node of the tree.
    Root(FileId, GreenId),
    /// A child node.
    Child {
        /// The parent of the node.
        parent: SyntaxStablePtrId,
        /// The SyntaxKind of the node.
        kind: SyntaxKind,
        /// A list of field values for this node, to index by.
        /// Which fields are used is determined by each SyntaxKind.
        /// For example, a function item might use the name of the function.
        key_fields: Vec<GreenId>,
        /// Chronological index among all nodes with the same (parent, kind, key_fields).
        index: usize,
    },
}
```

# Exercise 2: Lexer

Now we need to teach our lexer to recognize the new syntax, because we've added a new keyword.

**Teach the Cairo lexer to recognize the `caesar` keyword.**

Remember to update lexer tests and make sure they pass!

# Chapter 3: Testing framework

The compiler codebase makes extensive use of snapshot testing.

> Snapshot tests (also sometimes called approval tests) are tests that assert values against a reference value (the
> snapshot). This is similar to how `assert_eq!` lets you compare a value against a reference value but unlike simple
> string
> assertions, snapshot tests let you test it against complex values and come with comprehensive tools to review changes.
>
> Snapshot tests are particularly useful if your reference values are very large or change often.
>
> _~[`insta`](https://docs.rs/insta/latest/insta/#what-are-snapshot-tests) crate documentation_

`crates/cairo-lang-semantic/src/expr/test.rs`:

```rust
// This macro creates a set of #[test] functions that will run our snapshot tests.
cairo_lang_test_utils::test_file_test!(
    // Name of a module wrapping all test functions.
    expr_diagnostics,

    // Base path to snapshot fixtures.
    "src/expr/test_data",

    // List of test functions (pairs name-fixture file name).
    // Fixtures often have no file extension, but I recommend using *.txt for new test suites.
    {
        assignment: "assignment",
        attributes: "attributes",
        constant: "constant",
        // ...
    },

    // A name of a function that will run tests ("test runner").
    // For some complex use-cases you may create a stateful structure here,
    // but this is rare.
    test_function_diagnostics
);

// A little bit of magic is happening here.
// The runner function takes a set of `inputs`, sometimes also `args`
// and produces a `result` that mostly consists of an `output`.
// Fixtures are made out of several tests, each being composed of `sections`.
//
// All sections that exist in the fixture *at the moment of loading the test* are coming as `inputs`.
// The goal of the runner function is to produce a set of sections (`output`) to be compared with `inputs`.
// In test mode, the framework checks if `output` is a strict subset of `inputs` (asserting equality).
// In `CAIRO_FIX_TESTS=1` mode, output merged into input and the fixture file is overwritten.
//
// This has some important consequences:
// * You rarely `assert_eq!` in runner functions.
// * Renaming a section will probably make it stay there forever, unused, until the fixture is manually cleared.
// * Input sections are not visually different from expectation sections, so make sure you name your sections clearly.
pub fn test_function_diagnostics(
    inputs: &OrderedHashMap<String, String>,
    args: &OrderedHashMap<String, String>,
) -> TestRunnerResult {
    let db = &SemanticDatabaseForTesting::default();

    let diagnostics = setup_test_function_ex(
        db,
        inputs["function"].as_str(),
        inputs["function_name"].as_str(),
        inputs["module_code"].as_str(),
        inputs.get("crate_settings").map(|x| x.as_str()),
    )
        .get_diagnostics();
    let error = verify_diagnostics_expectation(args, &diagnostics);

    TestRunnerResult {
        outputs: OrderedHashMap::from([("expected_diagnostics".into(), diagnostics)]),
        error,
    }
}
```

`crates/cairo-lang-semantic/src/expr/test_data/attributes`:

```
//! > Test instantiation.       ← this is functioning just as a comment

//! > test_runner_name
                           ↓ everything here comes as `args`
test_function_diagnostics(expect_diagnostics: true)
     ↑ must match source code

         ↓ this is name of a section
//! > function
fn foo() {
    MyStruct {};
    MyEnum::a(3);
}

//! > function_name
foo

//! > module_code
#[phantom]
struct MyStruct {}

#[phantom]
enum MyEnum {
    a: felt252
}

//! > expected_diagnostics
error: Can not create instances of phantom types.
 --> lib.cairo:9:5
    MyStruct {};
    ^*********^

error: Can not create instances of phantom types.
 --> lib.cairo:10:5
    MyEnum::a(3);
    ^**********^


//! > ==========================================================================
         ↑ this is test separator
...
```

# Exercise 3: Parser

**Teach the Cairo parser to process `caesar(...)` expressions.**

Do not forget to add partial tree tests!
Try to think of as many edge cases as possible.

# Chapter 4: Salsa

> Salsa is a Rust framework for writing incremental, on-demand programs—these are programs that want to adapt to
> changes in their inputs, continuously producing a new output that is up-to-date. Salsa is based on the incremental
> recompilation techniques that we built for rustc, and many (but not all) of its users are building compilers or other
> similar tooling.
>
> ~Salsa Book

> [!WARNING]
> Cairo compiler is using Salsa `0.16.1`.
> This version is quite old, and the API has changed a lot since then.
> Especially, the hosted Salsa Book is now describing the total rewrite of Salsa: Salsa 3, formerly Salsa 2022.
>
> You can browse the old book here: <https://github.com/salsa-rs/salsa/tree/754eea8b5f8a31b1100ba313d59e41260b494225>

## Key idea

> The key idea of `salsa` is that you define your program as a set of
> **queries**. Every query is used like a function `K -> V` that maps from
> some key of type `K` to a value of type `V`. Queries come in two basic
> varieties:
>
> - **Inputs**: the base inputs to your system. You can change these
    whenever you like.
> - **Functions**: pure functions (no side effects) that transform your
    inputs into other values. The results of queries are memoized to
    avoid recomputing them a lot. When you make changes to the inputs,
    we'll figure out (fairly intelligently) when we can re-use these
    memoized values and when we have to recompute them.
>
> ~Salsa Book

## How to use Salsa in three easy steps

> Using Salsa is as easy as 1, 2, 3...
>
> 1. Define one or more **query groups** that contain the inputs
     and queries you will need. We'll start with one such group, but
     later on you can use more than one to break up your system into
     components (or spread your code across crates).
> 2. Define the **query functions** where appropriate.
> 3. Define the **database**, which contains the storage for all
     the inputs/queries you will be using. The query struct will contain
     the storage for all the inputs/queries and may also contain
     anything else that your code needs (e.g., configuration data).
>
> ~Salsa Book

## Interning

> **Interning** is a technique used to store only one copy of each distinct value, typically a string or other immutable
> data, to save memory and improve performance. When a value is interned, it is stored in a global or shared table, and
> subsequent requests for the same value return a reference to the existing entry rather than creating a new object.
> This
> ensures that identical values share the same memory location, reducing redundancy and enabling faster comparisons.
>
> ~LLM

> - We introduce `#[salsa::interned]` queries which convert a `Key` type into a numeric index of type `Value`,
    where `Value` is either the type `InternId` (defined by a salsa) or some newtype thereof.
> - Each interned query `foo` also produces an inverse `lookup_foo` method that converts back from the `Value` to
    the `Key` that was interned.
> - The `InternId` type (defined by salsa) is basically a newtype'd integer, but it internally uses `NonZeroU32` to
    enable space-saving optimizations in memory layout.
> - The `Value` types can be any type that implements the `salsa::InternIndex` trait, also introduced by this RFC. This
    trait has two methods, `from_intern_id` and `as_intern_id`.
>
> ~Intern Queries RFC

More info: <https://github.com/salsa-rs/salsa/blob/v0.17.0-pre.2/book/src/rfcs/RFC0002-Intern-Queries.md>

## Salsa use in Cairo compiler

Almost entirety of compiler machinery is written as Salsa queries and is encompassed into a `RootDatabase`.

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

Particular query groups resemble phases of compilation. Sorted topologically by query group dependencies:

1. `FilesGroup`:
    1. Stores all crucial **inputs** for the compiler:
        1. list of crates to compile and their configuration,
        2. overrides of file contents,
        3. `#[cfg]` item set,
        4. compilation flags (pretty dead concept nowadays).
    2. Provides queries for reading file contents.
2. `SyntaxGroup`:
    1. Interns AST nodes.
    2. Provides basic queries for AST traversal.
3. `ParserGroup`:
    1. Provides queries for parsing files (i.e. `FileId -> SyntaxNode`).
4. `DefsGroup`:
    1. Provides queries that extract unique identifiers of nameable items: modules, functions, constants, etc.
    2. Provides basic queries for traversing nameable item relations, for
       example `crate -> all its files`, `file -> modules`, `module -> submodules`, `module -> fns`.
    3. Compiler plugin/macro expansion is also hidden here.
5. `SemanticGroup`:
    1. Provides queries for actually analyzing the code.
6. `LoweringGroup`:
    1. The actual Cairo code and its semantic model are too complex to be directly optimized and used for generating
       Sierra code, queries here are responsible for doing appropriate transformations/preprocessing before doing
       further work.
7. `SierraGenGroup`:
    1. Actual Sierra code generation.

Here is a gist of queries code from the compiler,
this should be self-descriptive without all the clutter that exists in real code:

```rust
#[salsa::query_group(FilesDatabase)]
pub trait FilesGroup {
    #[salsa::interned]
    fn intern_crate(&self, crt: CrateLongId) -> CrateId;

    // ...

    /// Main input of the project. Lists all crates configurations.
    #[salsa::input]
    fn crate_configs(&self) -> Arc<OrderedHashMap<CrateId, CrateConfiguration>>;

    // ...

    /// Query for raw file contents. Private.
    fn priv_raw_file_content(&self, file_id: FileId) -> Option<Arc<String>>;
    /// Query for the file contents. This takes overrides into consideration.
    fn file_content(&self, file_id: FileId) -> Option<Arc<String>>;

    // ...
}

fn priv_raw_file_content(db: &dyn FilesGroup, file: FileId) -> Option<Arc<String>> {
    match file.lookup_intern(db) {
        FileLongId::OnDisk(path) => match fs::read_to_string(path) {
            Ok(content) => Some(Arc::new(content)),
            Err(_) => None,
        },
        FileLongId::Virtual(v) => Some(v.content),
    }
}

fn file_content(db: &dyn FilesGroup, file: FileId) -> Option<Arc<String>> {
    let overrides = db.file_overrides();
    overrides.get(&file).cloned().or_else(|| db.priv_raw_file_content(file))
}

#[salsa::query_group(SyntaxDatabase)]
pub trait SyntaxGroup: FilesGroup + Upcast<dyn FilesGroup> {
    #[salsa::interned]
    fn intern_green(&self, field: Arc<GreenNode>) -> GreenId;

    // ...

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
  into the incremental algorithm and explains (at a high level) how Salsa is
  implemented.

# Chapter 5: Semantic model

> The purpose of a "semantic model"
> in the Cairo compiler is to provide a structured representation of the program's meaning.
> It involves analyzing the source code to understand its structure,
> types, scopes and relationships between different parts of the code.
> This model is used for various tasks such as type checking, resolving identifiers,
> and ensuring that the code adheres to the language's rules and constraints.
> It detects semantic errors.
>
> ~LLM

The `SemanticGroup` is a big bag of invaluable queries that allow analyzing the code.
If you need to know anything about the code,
this is probably a good starting point for looking for a method of computing required info.

This is where type checking, name resolution and other semantic analysis are happening.

Queries of the `SemanticGroup` answer questions like the following:

1. _What does `x` refer to?_
2. _What are the members of struct `S`?_
3. _Is there am impl of trait `T` for enum `E`?_
4. _What is the type of expression `2 + 2`?_
5. _How function `foo<A, B, C>()` should be monomorphized at this usage place?_

# Exercise 4: Semantic model

**Implement semantic model for the `caesar(...)` expression.**

Remember, semantic model describes "meaning" of the code.
Do not encrypt the value here, just store input.

Don't forget about adding tests! Just a happy-path is fine for us here.

> [!TIP]
> You can copy quite a lot from a short string semantic model here.

> [!TIP]
> You will likely stumble upon a cryptic compilation error here.
> You will have to duplicate some line in some macro.

# Chapter 6: CairoLS

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

1. CairoLS ends on the semantic model.
2. It uses its own Salsa database, `AnalysisDatabase`, which has a subset of compiler queries, but adds some extra ones.
3. CairoLS is doing a lot of reverse queries, i.e., it is asking for things bottom-top, which is not happening in the
   compiler.
4. The usual flow is code location → AST → semantic model → analysis → semantic model → AST → code location.

# Chapter 7: Lowering

- Lowering is a process of transforming semantic models of functions into a _Control Flow Graph_ (CFG).
- CFG has a very simple, almost Sierra-like structure.
    - It's composed of _blocks_ (sometimes called _basic blocks_, for example, in LLVM).
        - Blocks are sequences of statements, which have _Single Static Assignment_ (SSA) form.
        - Blocks are connected with _jumps_ that can only exist at the end of the block.
- The CFG is a very good representation for optimization.

The following Cairo code:

```cairo
fn foo(a: bool, x: felt252) -> felt252 {
    if a {
        1
    } else {
        x
    }
}
```

Is lowered to:

```text
Parameters: v0: core::bool, v1: core::felt252
blk0 (root):
Statements:
End:
  Match(match_enum(v0) {
    bool::False(v2) => blk1,
    bool::True(v3) => blk2,
  })

blk1:
Statements:
End:
  Return(v1)

blk2:
Statements:
  (v4: core::felt252) <- 1
End:
  Return(v4)
```

# Exercise 5: Lowering

**Lower caesar expressions to encrypted literals in the lowering model.**

Add a test to verify your solution works!

With 3R Caesar cipher, the short string `'foobar'` (`112628796121458` as felt) encrypts to `'irredu'` (`115940266435701`
as felt).

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

# Chapter 8: Sierra generation in a nutshell

- This is implemented in the `SierraGen` Salsa group.
- But! I have a happy message for all of you!
- I don't know how it works, and you don't need to know it either. 🥳

# Exercise 6: The grand finale: an end-to-end test

Congratulations! You've made it to the final exercise.
This one is trivial but very satisfying.

**Add an E2E test for your new syntax.**
