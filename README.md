[@@deriving]
============

_deriving_ is a library that simplifies type-driven code generation that works on OCaml >=4.02.

_deriving_ includes a set of useful plugins: Show, Eq, Ord, [Protobuf][].

[protobuf]: https://github.com/whitequark/ppx_deriving_protobuf

Installation
------------

_deriving_ can be installed via [OPAM](https://opam.ocaml.org):

    opam install ppx_deriving

Usage
-----

From a user's perspective, _deriving_ is triggered by a `[@@deriving Plugin]` annotation attached to a type declaration in structure or signature:

``` ocaml
type point2d = float * float
[@@deriving Show]
```

It's possible to invoke several plugins by separating their names with commas:

``` ocaml
type point3d = float * float * float
[@@deriving Show, Eq]
```

It's possible to pass options to a plugin by appending a record to plugin's name:

``` ocaml
type t = string
[@@deriving Ord { affix = "" }]
```

It's possible to make _deriving_ ignore a missing plugin rather than raising an error by passing an `optional = true` option (TBD), for example, to enable conditional compilation:

``` ocaml
type addr = string * int
[@@deriving Json { optional = true }]
```

### Working with existing types

At first, it may look like _deriving_ requires complete control of the type declaration. However, a lesser-known OCaml feature allows to derive functions for any existing type. Using `Pervasives.fpclass` as an example, _Show_ can be derived as follows:

``` ocaml
# module M = struct
  type fpclass = Pervasives.fpclass = FP_normal | FP_subnormal | FP_zero | FP_infinite | FP_nan
  [@@deriving Show]
end;;
module M :
  sig
    type fpclass =
      fpclass =
        FP_normal
      | FP_subnormal
      | FP_zero
      | FP_infinite
      | FP_nan
    val pp_fpclass : Format.formatter -> fpclass -> unit
    val show_fpclass : fpclass -> bytes
  end
# M.show_fpclass FP_normal;;
- : bytes = "FP_normal"
```

The module is used to demonstrate that `show_fpclass` really accepts `Pervasives.fpclass`, and not just a shadowing, identically named type.

The need to repeat the type definition may look tedious, but consider this: if the definition was automatically imported from the declaration point, how would you attach attributes to refine the behavior of the deriving plugin?

Plugin conventions
------------------

It is expected that all _deriving_ plugins will follow the same conventions, thus simplifying usage.

  * By default, the functions generated by a plugin for a `type foo` are called `fn_foo` or `foo_fn`. The defaults can be overridden by an `affix` plugin option.

  * There may be additional attributes attached to the AST. In case of a plugin named `Eq` and attributes named `compare` and `skip`, the plugin must recognize all of `compare`, `skip`, `eq.compare`, `eq.skip`, `deriving.eq.compare` and `deriving.eq.skip` annotations. However, if it detects that at least one namespaced (e.g. `eq.compare` or `deriving.eq.compare`) attribute is present, it must not look at any attributes located within a different namespace. As a result, different plugins can avoid interference even if they share attribute names.

  * A typical plugin should handle tuples, records, normal and polymorphic variants; builtin types: `int`, `int32`, `int64`, `nativeint`, `float`, `bool`, `char`, `string`, `bytes`, `list`, `array`, `option` and their `Mod.t` aliases; and abstract types. For builtin types, it should have customizable, sensible default behavior. For abstract types, it should expect to find the functions it would derive itself for that type.

  * If a type is parametric, the generated functions accept an argument for every type variable before all other arguments.

Plugin: Show
------------

_Show_ derives a function that inspects a value; that is, pretty-prints it with OCaml syntax. However, _Show_ offers more insight into the structure of values than the Obj-based pretty printers (e.g. `Printexc`), and more flexibility than the toplevel printer.

``` ocaml
# type t = [ `A | `B of int ] [@@deriving Show];;
type t = [ `A | `B of i ]
val pp_t : Format.formatter -> [< `A | `B of i ] -> unit = <fun>
val show_t : [< `A | `B of i ] -> bytes = <fun>
# show_t (`B 1);;
- : bytes = "`B 1"
```

For abstract type `t`, _Show_ expects to find a `pp_t` function in the corresponding module.

_Show_ allows to specify custom formatters for types to override default behavior. A formatter for type `t` has a type `Format.formatter -> t -> unit`:

``` ocaml
# type file = {
  name : string;
  perm : int     [@printer fun fmt -> Format.fprintf fmt "0o%03o"];
} [@@deriving Show];;
# show_file { name = "dir"; perm = 0o755 };;
- : bytes = "{ name = \"dir\"; perm = 0o755 }"
```

Plugins: Eq and Ord
-------------------

_Eq_ derives a function comparing values by semantic equality; structural or physical depending on context. _Ord_ derives a function defining a total order for values, returning `-1`, `0` or `1`. They're similar to `Pervasives.(=)` and `Pervasives.compare`, but are faster, allow to customize the comparison rules, and never raise at runtime. _Eq_ and _Ord_ are short-circuiting.

``` ocaml
# type t = [ `A | `B of int ] [@@deriving Eq, Ord];;
type t = [ `A | `B of int ]
val equal_t : [> `A | `B of int ] -> [> `A | `B of int ] -> bool = <fun>
val compare_t : [ `A | `B of int ] -> [ `A | `B of int ] -> int = <fun>
# equal_t `A `A;;
- : bool = true
# equal_t `A (`B 1);;
- : bool = false
# compare_t `A `A;;
- : int = 0
# compare_t (`B 1) (`B 2);;
- : int = -1
```

For variants, _Ord_ uses the definition order. For builtin types, properly monomorphized `(=)` is used for _Eq_, or corresponding `Mod.compare` function (e.g. `String.compare` for `string`) for _Ord_. For abstract type `t`, _Eq_ and _Ord_ expect to find an `equal_t` or `compare_t` function in the corresponding module.

_Eq_ and _Ord_ allow to specify custom comparison functions for types to override default behavior. A comparator for type `t` has a type `t -> t -> bool` for _Eq_ or `t -> t -> int` for _Ord_. If an _Ord_comparator returns a value outside -1..1 range, the behavior is unspecified.

``` ocaml
# type file = {
  name : string [@equal fun a b -> String.(lowercase a = lowercase b)];
  perm : int    [@compare fun a b -> compare b a]
} [@@deriving Eq, Ord];;
type file = { name : bytes; perm : int; }
val equal_file : file -> file -> bool = <fun>
val compare_file : file -> file -> int = <fun>
# equal_file { name = "foo"; perm = 0o644 } { name = "Foo"; perm = 0o644 };;
- : bool = true
# compare_file { name = "a"; perm = 0o755 } { name = "a"; perm = 0o644 };;
- : int = -1
```

Developing plugins
------------------

This section only explains the tooling and best practices. Anyone aiming to implement their own deriving plugin is encouraged to explore the existing ones, e.g. [Eq](src_plugins/ppx_deriving_eq.ml) or [Show](src_plugins/ppx_deriving_show.ml).

### Tooling and environment

A deriving plugin is expected packaged as a Findlib library. When encountering a statement of form `[@@deriving Foo]`, _deriving_ searches for library `ppx_deriving_foo` (the lowercase of plugin name) and then [dynlinks](http://caml.inria.fr/pub/docs/manual-ocaml/libref/Dynlink.html) the module `ppx_deriving_foo.cmo` (or `.cmxs`). The library should have a Findlib dependency on `ppx_deriving.api`.

The module must register itself using `Ppx_deriving.register "Foo"` during loading. The module must register one and only one deriver.

It is possible to test the plugin without installing it by instructing _deriving_ to load it directly; the compiler should be invoked as `ocamlfind c -ppx 'ppx_deriving src/ppx_deriving_foo.cmo' ...`. The file extension is replaced with `.cmxs` automatically for native builds. This can be integrated with buildsystem, e.g.: [myocamlbuild.ml](myocamlbuild.ml#L9-L13).

### Goals of the API

_deriving_ is a thin wrapper over the ppx extension system. Indeed, it includes very little logic; the goal of the project is 1) to provide common reusable abstractions required by most, if not all, deriving plugins, and 2) encourage the deriving plugins to cooperate and to have as consistent user interface as possible.

As such, _deriving_:

  * Completely defines the syntax of `[@@deriving]` annotation and unifies the plugin discovery mechanism;
  * Provides an unified, strict option parsing API to plugins (TBD);
  * Provides helpers for parsing annotations to ensure that the plugins interoperate with each other and the rest of the ecosystem.

### Using the API

Complete API documentation is available [online](http://whitequark.github.io/ppx_deriving/Ppx_deriving.html).

The following is a list of tips for developers trying to use the ppx interface:

  * Module paths overwhelm you? Open all of the following modules, they don't conflict with each other: `Longident`, `Location`, `Asttypes`, `Parsetree`, `Ast_helper`, `Ast_convenience`.
  * Need to insert some ASTs? See [ppx_metaquot](https://github.com/alainfrisch/ppx_tools/blob/master/ppx_metaquot.ml); it's required by `ppx_deriving.api`.
  * Need to display an error? Use `Ppx_deriving.raise_errorf ~loc "Cannot derive Foo: (error description)"` ([doc](http://whitequark.github.io/ppx_deriving/Ppx_deriving.html#VALraise_errorf)); keep it clear which deriving plugin raised the error!
  * Need to derive a function name from a type name? Use [Ppx_deriving.mangle_lid](http://whitequark.github.io/ppx_deriving/Ppx_deriving.html#VALmangle_lid).
  * Need to fetch an attribute from a node? Use `Ppx_deriving.attr ~prefix "foo" nod.nod_attributes` ([doc](http://whitequark.github.io/ppx_deriving/Ppx_deriving.html#VALattr)); this takes care of interoperability.
  * Put all functions derived from a set of type declarations into a single `let rec` block; this reflects the always-recursive nature of type definitions.
  * Need to handle polymorphism? Use [Ppx_deriving.poly_fun_of_type_decl](http://whitequark.github.io/ppx_deriving/Ppx_deriving.html#VALpoly_fun_of_type_decl) for derived functions, [Ppx_deriving.poly_arrow_of_type_decl](http://whitequark.github.io/ppx_deriving/Ppx_deriving.html#VALpoly_arrow_of_type_decl) for signatures, and [Ppx_deriving.poly_apply_of_type_decl](http://whitequark.github.io/ppx_deriving/Ppx_deriving.html#VALpoly_apply_of_type_decl) for "forwarding" the arguments corresponding to type variables to another generated function.
  * Need to apply a sequence or a binary operator to variant, tuple or record elements? Use [Ppx_deriving.fold_exprs](http://whitequark.github.io/ppx_deriving/Ppx_deriving.html#VALfold_exprs).
  * Don't forget to invoke the option parser (TBD) even if you don't have any options. This way, it would display an error to the user.
  * Your structure generator is throwing an exception that prevents you from returning valid output for signatures? Use `try .. with Location.Error err -> [Str.extension (Ast_mapper.extension_of_error err)]` to convert it to a pure representation.

### Dynlink rationale

_deriving_ is using Dynlink. This can be seen as controversional; here are the reasons for it being a superior solution:

  * Dynlink allows to combine completely automatic discovery of plugins in both toplevel and batch compilation with strict control over annotation syntax. It is not currently possible to amend a ppx command line by requiring another package in batch compilation, and it will never be possible to do that in toplevel.
  * Having a single ppx responsible for `[@@deriving]` annotation means it's possible to error out when a deriving plugin is missing. It's still possible to forget to include `ppx_deriving` as a whole, but less likely so.
  * Having a single ppx that processes `[@@deriving]` annotation allows strict control over syntax.
  * Having a single ppx that processes `[@@deriving]` means that there is much less forking, the code doesn't pay for plugins it doesn't use, the plugins can stay plentiful & small, and all the ASTs are traversed exactly once.
  * After all else fails, or if the overhead of Dynlink is too high, it's possible to simply link `ppx_deriving_main.cmxa` with the required plugins without any additional OCaml code to get a combined executable.

The only argument against Dynlink or the point single responsibility I can think of is an inability to write a deriving plugin completely independent from any other library. I do not see a reason this would be desirable.

License
-------

_deriving_ is distributed under the terms of [MIT license](LICENSE.txt).
