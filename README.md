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

TODO

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
    Node { children: Vec<GreenId>, width: TextWidth },
}
```

- Green nodes is an AST representation that is not strictly typed:
- It has no accessors, it's for example hard to find out function name in function definition subtree.
- But it is trivial to traverse in various directions and to manipulate, e.g. in formatter.

## Typed syntax tree

```rust
pub trait TypedSyntaxNode {
    const OPTIONAL_KIND: Option<SyntaxKind>;
    type StablePtr: TypedStablePtr;
    type Green;
    fn missing(db: &dyn SyntaxGroup) -> Self::Green;
    fn from_syntax_node(db: &dyn SyntaxGroup, node: SyntaxNode) -> Self;
    fn as_syntax_node(&self) -> SyntaxNode;
    fn stable_ptr(&self) -> Self::StablePtr;
}
```

- On top of green (sub)tree, a typed AST can be built.
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

Do not forget to update lexer tests and make sure they pass!

# Chapter 3: Testing framework

`crates/cairo-lang-semantic/src/expr/test.rs`:

```rust
cairo_lang_test_utils::test_file_test!(
    expr_diagnostics,
    "src/expr/test_data",

    {
        assignment: "assignment",
        attributes: "attributes",
        constant: "constant",
        // ...
    },
    test_function_diagnostics
);

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
//! > Test instantiation.

//! > test_runner_name
test_function_diagnostics(expect_diagnostics: true)

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

...
```

# Exercise 3: Parser

**Teach the Cairo parser to process `caesar(...)` expressions.**

Do not forget to add partial tree tests!
Try to think of as many edge cases as possible.

# Chapter 4: Salsa

> Salsa is a Rust framework for writing incremental, on-demand programs -- these are programs that want to adapt to
> changes in their inputs, continuously producing a new output that is up-to-date. Salsa is based on the incremental
> recompilation techniques that we built for rustc, and many (but not all) of its users are building compilers or other
> similar tooling.

> [!WARNING]
> Cairo compiler is using Salsa `0.16.1`.
> This version is quite old and the API has changed a lot since then.
> Especially, the hosted Salsa Book is now describing the total rewrite of Salsa: Salsa 3, formerly Salsa 2022.
>
> You can browse the old book here: <https://github.com/salsa-rs/salsa/tree/754eea8b5f8a31b1100ba313d59e41260b494225>

## Key idea

The key idea of `salsa` is that you define your program as a set of
**queries**. Every query is used like a function `K -> V` that maps from
some key of type `K` to a value of type `V`. Queries come in two basic
varieties:

- **Inputs**: the base inputs to your system. You can change these
  whenever you like.
- **Functions**: pure functions (no side effects) that transform your
  inputs into other values. The results of queries are memoized to
  avoid recomputing them a lot. When you make changes to the inputs,
  we'll figure out (fairly intelligently) when we can re-use these
  memoized values and when we have to recompute them.

## How to use Salsa in three easy steps

Using Salsa is as easy as 1, 2, 3...

1. Define one or more **query groups** that contain the inputs
   and queries you will need. We'll start with one such group, but
   later on you can use more than one to break up your system into
   components (or spread your code across crates).
2. Define the **query functions** where appropriate.
3. Define the **database**, which contains the storage for all
   the inputs/queries you will be using. The query struct will contain
   the storage for all the inputs/queries and may also contain
   anything else that your code needs (e.g., configuration data).

## Salsa use in Cairo compiler

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
#[salsa::query_group(FilesDatabase)]
pub trait FilesGroup {
    #[salsa::interned]
    fn intern_crate(&self, crt: CrateLongId) -> CrateId;
    
    // ...

    /// Main input of the project. Lists all the crates configurations.
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

1. CairoLS ends on the semantic model.
2. It uses its own Salsa database, `AnalysisDatabase`, which has a subset of compiler queries, but adds some extra ones.
3. CairoLS is doing a lot of reverse queries, i.e. it is asking for things bottom-top, which is not happening in the compiler.
4. The usual flow is code location → AST → semantic model → analysis → semantic model → AST → code location. 

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

# Chapter 6: Sierra generation in a nutshell

- This is implemented in the `SierraGen` Salsa group.
- But! I have a happy message for all of you!
- I don't know how it works, and you don't need to know it either. 🥳

# Exercise 6: The grand finale: an end-to-end test

Congratulations! You've made it to the final exercise.
This one is trivial but very satisfying.

**Add an E2E test for your new syntax.**

---

# Appendix: Credits

Parts of Salsa information is copied verbatim from the Salsa Book.
