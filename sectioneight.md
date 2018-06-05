# Section 8

## Lexical Environments

A `Lexical Environment` is a specification value, which means it is an abstract concept, used to explain how the ES language internally works, but that doesn't actually correspond to a concrete entity in an implementation of the language.

A `Lexical Environment` keeps track of the bindings (identifier name -> value) within a certain scope.

A `Lexical Environment` is formed of 2 parts:
- `Environment Record`, which is what actually holds the bindings
-  outer environment reference, which is a pointer to another `Lexical Environment`

When we try access a certain binding, first the `Environment Record` for the `Lexical Environment` we are in is consulted. If this `Environment Record` does not hold that binding, then the outer environment reference will point us to the next `Lexical Environment` in which we can execute the look-up. And so on, until either we find the binding or we get to the global `Lexical Environment` (what happens next depends on what kind of look-up is being executed and in which mode the code is running).
This series of `Lexical Environment` "links" is what forms the scope chain.

## Environment Records

An `Environment Record` is what actually holds the bindings of a `Lexical Environment`.

There are 2 main kinds of `Environment Record`:
- declarative `Environment Record`
- object `Environment Record`

A declarative `Environment Record` can either be a function `Environment Record` or a module `Environment Record`.

Finally, a global `Environment Record` is both a declarative `Environment Record` and an object `Environment Record`.

Each `Environment Record` must have some abstract operations on it, but based on the kind of `Environment Record` the actual algorithms for these operations can be different.

- `HasBinding(N) -> boolean`: determine if the `Environment Record` has a binding for the string value `N`.

- `CreateMutableBinding(N, D)`: create a new, non-initialized mutable binding in the `Environment Record`. The string `N` is the name of the identifier. `D` is a boolean and if it is **true** then this binding can be deleted.

- `CreateImmutableBinding(N, S)`: create a new, non-initialized immutable binding. The string `N` is the identifier name. `S` is a boolean and if it is **true** then attempts to assign a new value to this binding will throw an exception.

- `InitializeBinding(N, V)`: set the value of an existing but non-initialized binding with the name of `N`. `V` is the value to be assigned and can be any a value of any type.

- `SetMutableBinding(N, V, S)`: set the value of an existing mutable binding with the name of `N`. `V` is the value to be assigned. `S` is a boolean. If it is **true** and the binding `N` cannot be set, then throw a `TypeError`.

- `GetBindingValue(N, S) -> any`: return the value of an existing binding with the name of `N`. `S` is a boolean and if it is **true** and the binding does not exist throw a `ReferenceError`. If the binding exists but is not initialized then a `ReferenceError` is always thrown.

- `DeleteBinding(N) -> boolean`: delete a binding with the name `N`. If `N` is successfully deleted or if it doesn't exists, then return **true**. If `N` can't be deleted, return **false**.

- `HasThisBinding() -> boolean`: return **true** if the `Environment Record` has a `this` binding. Else **false**.

- `HasSuperBinding() -> boolean`: return **true** if the `Environment Record` has a `super` binding. Else **false**.

- `WithBaseObject() -> object|undefined`: if the `Environment Record` is associated with a `with` object, then return that object. Else return **undefined**.

**Note:** `GetBindingValue` tries to retrieve the value for a certain binding `N`. If such binding exists, but is *not initialized*, then the operation throws a `Reference Error`. A case for a binding not being initialized is when we declare a variable by using the `let` or `const` keywords. The binding for this variable will be added to the `Environment Record` as soon as the `Environment Record` is created, but such binding will be non-initialized. The initialization happens only when the code is being executed and the actual variable declaration is met. That's why we can't access `let` and `const` variables before the line that declares them. This is not the case for variables declared by the `var` keyword. They are in fact immediately initialized to the **undefined** value (aka *hosting*).

### Declarative Environment Record

A declarative `Environment Record` is created for a `Lexical Scope` associated with modules, functions invocation and blocks of code.

The bindings of a declarative `Environment Record` directly associate identifier names with values.

#### Function Environment Record

A function `Environment Record` is a declarative `Environment Record` associated with the invocation of a function.

All the identifiers (variables, functions, parameters) declared in the body of the function are added as bindings on the `Environment Record`.

A function `Environment Record` also has a binding for the `this` and `super` keywords.

**Note on closures:**

The outer environment reference for the `Lexical Environment` associated with a function invocation has the value of the function `[[Environment]]` internal slot. This is important to notice, because the value of this internal slot is assigned when the function is initialized and it depends on **where the function declaration exists**.

Let's say we have a function `a`. When this function is invoked, a corresponding `Lexical Environment` is created. All the identifiers declared inside `a` are added as bindings on the function `Environment Record`. Imagine that among these identifiers there is one for a function `b`. When the function `b` is initialized its `[[Environment]]` internal slot is set to be the `Lexical Environment` for `a`.

Later, the function `b` will be invoked and a new `Lexical Environment` is generated. The outer environment reference for this `Lexical Environment` is set to the the value of `b.[[Environment]]`. As we know, the outer environment reference is used every time a look-up for an identifier fails, to repeat such look-up in the next `Lexical Environment`. This means that because of that fact that `b` was **declared inside `a`**, `b` always has access to the bindings of `a`.

#### Module Environment Record

A module `Environment Record` is a declarative `Environment Record` associated with a module.

Its bindings are all the identifiers declared inside the module, but also all the bindings imported from other modules.

The outer environment reference for the `Lexical Environment` of a module `Environment Record` is always the global `Lexical Environment`.

### Object Environment Record

An object `Environment Record` associates its bindings with the property of a certain object, called its *binding object*.

This means that all the properties of the binding object (own, inherited, enumerable and non-enumerable) are added as bindings on this `Environment Record`.

Whenever a new binding is created in the `Environment Record` a corresponding property is added to the binding object. When a new property is added to the binding object a corresponding binding is created in the `Environment Record`.

An object `Environment Record` can be generated by the `with` keyword.

### Global Environment Record

A global `Environment Record` is a single `Record` but it contains both a global `Environment Record` and a declarative `Environment Record`.

The binding object associated with its object `Environment Record` is the *global object*. The global object provides initial bindings that are always available in an ES program.

All the identifiers declared by using the `var` and `function` keywords are added as bindings on the object `Environment Record` (and thus as properties on the global object). Identifiers declared by using the `let` and `const` keywords are instead added as bindings on the declarative `Environment Record`.

### GetIdentifierReference(*lex, name, strict*)

This abstract operation is invoked with arguments `lex`, which is either **null** or a `Lexical Environment`, `name` which is a string representing the name of a certain identifier, and `strict` which is a boolean flag for whether the code is running in "strict mode" or not.

It tries to retrieve a `Reference` for the identifier `name`. It starts the look-up in the `Environment Record` of the `Lexical Environment` `lex`.

If this `Environment Record` does have a binding for `name`, then a new `Reference` is created, passing the `Environment Record` as its base value component, `name` as its referenced name component and `strict` as its strict reference flag.

Else, the algorithm retrieves the outer environment reference of `lex` and executes a new look-up in there.

If eventually the scope chain is exhausted, an **undefined** `Reference` is created and returned.

## Execution Contexts

An `Execution Context` is a concept associated with the execution of code.

Anytime there is new executable code, an `Execution Context` for that code is created. The `Execution Context` contains all the state necessary to run and pause/resume the code. The `Execution Context` knows the function (if any) that started the execution of the code and also the script/module in which the code lives.

There can only be one `Execution Context` actually executing code, which is called the running `Execution Context`. `Execution Context`s are organized in a stack, so that the `Execution Context` on top of the stack is the first to be removed (terminate execution).

When the running `Execution Context` meets new executable code (i.e. a function invocation), then a new `Execution Context` is created for that code, put on top of the stack and executed. After it terminates, the previous `Execution Context` resumes execution from the point at which it had paused.

Every ES program has a global `Execution Context`, then modules and function invocations also generate a new `Execution Context`.

An `Execution Context` contains a `Lexical Environment`. Identifiers look-ups start from the `Lexical Environment` of the running `Execution Context` and then use its outer environment reference to traverse the scope chain as necessary.

The value for `this` is also retrieved in a similar way. The scope chain is traversed until an `Environment Record` for which the `HasThisBinding` abstract operation returns **true** is found. There is always at least one `Environment Record` that provides a value for `this` and that is the global `Environment Record`.

It might happen that the `Lexical Environment` of the running `Execution Context` is temporarily replaced. For example, if a block of code is met inside the program, then a new `Lexical Environment` is created for that block. This block `Lexical Environment` has an outer environment reference to the `Lexical Environment` of the running `Execution Context`, then it actually replaces it inside the `Execution Context` for as long as the code of the block is executed.

**Note:** The execution of code has 2 phases:
1. First the `Execution Context` for that code must be created. This means creating a new `Lexical Environment` and a new `Environment Record`, retrieve all the identifiers declared in the scope of the code and create bindings for them inside the `Environment Record`, finally push the `Execution Context` on top of the stack.
2. Then, the code can actually be executed using all the information stored in its `Execution Context`.

## Jobs and Job Queues

`Job`s are related with asynchrony and where introduced in the ES language with ES6.

A `Job` is an abstract operation that rather than being executed right away, is scheduled for a "later" time.

This "later" time is determined by a `Job Queue`. `Job`s are pushed inside a queue and when the `Execution Context` stack is empty and there is no running `Execution Contex` the first `Job` from the queue is retrieved, a new `Execution Context` is created for this `Job`, pushed in the stack and executed.

An ES implementation might defined several `Job Queue`s, but it has to at least define the ScriptJobs `Job Queue`, which contains `Job`s related with script and module operations, and the PromiseJobs `Job Queue`, which contains `Job`s that handle the settling of Promises.

It's up to the ES implementation to determine what happens to a program when both the `Execution Context` stack and all the `Job Queue`s are empty.

To schedule a `Job` a `Record` for that `Job` is created. This takes the name of `PendingJob`. A `PendingJob` holds all the information necessary to correctly execute its `Job`, like the arguments `List` and the `Execution Context` that was running when the specific `PendingJob` was created. The `PendingJob` is what is actually pushed inside a `Job Queue`. `PendingJob`s are created by the `EnqueueJob` abstract operation.
