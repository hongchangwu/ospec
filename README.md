# OSpec

## Introduction

OSpec is a Behavior-Driven Development tool for OCaml, inspired by RSpec, a
Ruby BDD library. It is implemented as a Camlp4 syntax extension.

This is a work in progress and should be considered beta quality.

Note: OSpec requires OCaml >= 3.10.0 to run. On versions below 3.11.0, though,
there are two limitations due to OCaml toplevel bugs:

* If you want to use helpers in your specifications (see below), you must
  explicitely "open Helpers" at the top of every specification file.

* Running the "ospec" command with multiple files as arguments doesn't work.

## Usage

To build and install OSpec, simply type

```
$ ocaml setup.ml -configure
$ ocaml setup.ml -build
# ocaml setup.ml -install
```

These commands (and OSpec itself) rely on findlib, so you need to have it
installed for them to work. An executable called "ospec" which takes
specification files as command line arguments will be build. For example, the
command below will run the specifications in the file specs.ml:

```
$ ospec specs.ml
```

The default report format is "nested". You can choose the format from the
command line using the "-format" option. Currently the available formats are
"nested" and "progress".

## Syntax

Specifications are defined with the "describe" keyword. Inside a specification,
examples are defined, with the "it" keyword, including one or more expectations.
Expectations are tests comparing some result to an expected value using the
"should" keyword.

Here's useless specification which shows how OSpec's syntax works:

```ocaml
describe "The number one" do
  it "should equal 2 when added to itself" do
    (1 + 1) should = 2  (* anything 'a -> 'a -> bool should work *)
  done;

  it "should be positive" do
    let positive x = x > 0 in
    1 should be positive  (* 'a -> bool should work too *)
  done;

  it "should be negative when multiplied by -1" do
    let x = 1 * (-1) in
    x should be < 0;      (* "be" is optional *)
    x should not be >= 0
  done;

  it "should fail when divided by 0" do
    (* For exception tests, wrap it in a fun *)
    let f = fun () -> 1 / 0 in
    f should raise_an_exception;
    f should raise_exception Division_by_zero;
    f should not raise Exit
  done;

  it "should match ^[0-9]+$ when converted to a string" do
    (string_of_int 1) should match_regexp "^[0-9]+$"
  done;

  (* Specify behaviors still not implemented like this *)
  it "should be cool"
done
```

It is also possible to group related examples in nested specifications. See
examples/nested.ml for a sample.

## Before and After Blocks

OSpec supports "before" and "after" blocks, which can be used to run code
before or after running the examples. This is only useful for operations
which cause side-effects on some global variable (see examples/hooks.ml).
Defining a variable in a "before" block doesn't make it available for the
examples, since the scope of such a variable would be the "before" block
itself.

```ocaml
describe "An example" do
  before all do
    (* Code here runs once, before all examples. *)
  done;

  before each do
    (* Code here runs before each example. *)
  done;

  after each do
    (* Code here runs after each example. *)
  done;

  after all do
    (* Code here runs once, after all examples. *)
  done;

  it "should behave in a certain way" do
    (* ... *)
  done
done
```

## Match Syntax Extension

OSpec also extends the "match" syntax so that you can write expectations like

```ocaml
[1] should match h::t
[1] should not match []
```

## Property Testing

OSpec provides support for property testing with the "forall" function.
Combining forall with a sample generator (either one of the provided generators
or a custom one), it is possible to write specifications such as

```ocaml
describe "A list" do
  it "should equal itself when reversed twice" do
    forall (list_of int) l . (List.rev (List.rev l)) should = l
  done
done
```

In the example above, two generators are used: "list_of" and "int". The former
is a higher-order generator, since it takes another sample generator as a
parameter. This generator is used to sample values which the random list will
contain (in this case, int values). Each sample list will be bound to "l",
which can then be used in the property specified after the dot.

It is also possible to specify constraints to the generated samples, such as
in the (contrived) example below.

```ocaml
describe "A bool" do
  it "should be true if all samples are true" do
    forall bool b . b = true -> b should = true
  done
done
```

If the value of the boolean expression before the arrow is false for a given
sample, it is discarded and a new one is generated.

By default, 100 samples are generated for each property test. A different
number of samples may be specified explicitly as in the example below.

```ocaml
forall 42 bool b . b = true -> b should = true
```

This will generate 42 bool instances instead of the default 100.

## Predefined Generators

Below is a list of sample generators defined by OSpec. They can be used as
building blocks for custom generators for more complex data types. A generator
is simply a function of type (unit -> 'a). By convention, higher-order
generators are named with an "_of" suffix.

```ocaml
val bool : unit -> bool
  Generates true or false with 50% probabilty each.

val float : unit -> float
  Generates a random float in the interval [0, 1).

val int : unit -> int
  Generates a random int between 0 (inclusive) and max_int (exclusive).

val int_in_range : int -> int -> unit -> int
  Generates a random int in the inclusive range given by the two int arguments.

val int32 : unit -> Int32.t
  Generates a random int32 between 0 (inclusive) and Int32.max_int (exclusive).

val int32_in_range : int32 -> int32 -> unit -> int32
  Generates a random int32 in the inclusive range given by the two int32
  arguments.

val int64 : unit -> Int64.t
  Generates a random int64 between 0 (inclusive) and Int64.max_int (exclusive).

val int64_in_range : int64 -> int64 -> unit -> int64
  Generates a random int64 in the inclusive range given by the two int64
  arguments.

val nativeint : unit -> Nativeint.t
  Generates a random nativeint between 0 (inclusive) and Nativeint.max_int
  (exclusive).

val nativeint_in_range : nativeint -> nativeint -> unit -> nativeint
  Generates a random nativeint in the inclusive range given by the two nativeint
  arguments.

val char : unit -> char
  Generates a random character.

val char_in_range : char -> char -> unit -> char
  Generates a random character in the inclusive range given by the two char
  arguments.

val ascii : unit -> char
  Generates a random ASCII character.

val digit : unit -> char
  Generates a random digit character.

val lowercase : unit -> char
  Generates a random lowercase character [a-z].

val uppercase : unit -> char
  Generates a random uppercase character [A-Z].

val alphanumeric : unit -> char
  Generates a random alphanumeric character.

val string_of : ?length:(unit -> int) -> (unit -> char) -> unit -> string
  Given one of the character generators, returns a random string of length
  given by the "length" int generator.

val string : ?length:(unit -> int) -> unit -> string
  Generates a random string of ASCII characters. It is equivalent to
  "string_of ascii".

val list_of : ?length:(unit -> int) -> (unit -> 'a) -> unit -> 'a list
  Given a generator for the list elements, returns a random list of length
  given by the "length" int generator.

val list : ?length:(unit -> int) -> unit -> unit list
  Generates a random-sized list of unit elements.

val array_of : ?length:(unit -> int) -> (unit -> 'a) -> unit -> 'a array
  Given a generator for the array elements, returns a random array of length
  given by the "length" int generator.

val array : ?length:(unit -> int) -> unit -> unit array
  Generates a random-sized array of unit elements.

val queue_of : ?length:(unit -> int) -> (unit -> 'a) -> unit -> 'a Queue.t
  Given a generator for the queue elements, returns a random queue of length
  given by the "length" int generator.

val queue : ?length:(unit -> int) -> unit -> unit queue
  Generates a random-sized queue of unit elements.

val stack_of : ?length:(unit -> int) -> (unit -> 'a) -> unit -> 'a Stack.t
  Given a generator for the stack elements, returns a random stack of length
  given by the "length" int generator.

val stack : ?length:(unit -> int) -> unit -> unit stack
  Generates a random-sized stack of unit elements.

val hashtbl_of : ?length:(unit -> int) -> (unit -> 'a) * (unit -> 'b) -> unit ->
                 ('a, 'b) Hashtbl.t
  Given a tuple of generators for the hash table keys and values, returns a
  random hash table of length given by the "length" int generator.
```

## Helpers

Some helper functions are provided, as shown above. They are described below.

```ocaml
val raise_an_exception : (unit -> 'a) -> bool
  Returns true if any exception is raised

val raise_exception : (unit -> 'a) -> exn -> bool
  Returns true if the given exception is raised

val match_regexp : string -> string -> bool
  Returns true if the given string (first argument) matches the given regex
  (second argument).
```

## TODO

* Provide more report formats.
* Provide more helpers.
* Cleanup the code, which is horrible and hacky.
* ...
