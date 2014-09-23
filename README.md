rs-persistent-datastructures
============================

Mostly a Hash Array Mapped Trie implementation based on the
[Ideal Hash Trees](http://lampwww.epfl.ch/papers/idealhashtrees.pdf) paper by Phil Bagwell.
This is the persistent map datastructure used in Scala's and Clojure's standard libraries.
The idea to use a special *collision node* to deal with hash collisions is taken from Clojure's
implementation.

## Usage
```rust
let mut map = HamtMap::new();

for i in range(0, size) {
    map = map.plus(i, i);
}

if map.find(&0) == Some(1) {
    ...
}

let (without_10, size_changed_10) = map.remove(&10);
let (without_20, size_changed_20) = map.remove(&20);

for (k, v) in map.iter() {
    ...
}

```

## Performance
Looks pretty good so far, for a fully persistent data structure. The benchmarks below were done on
a Core i5 2400, with random numbers and the compile flags `-O --test -Zlto -C target-cpu=corei7-avx`.
I also turned off (commented out) the assertions in the code, which should not be necessary in a
release build.

### Lookup
Times (in microseconds) for one thousand lookups in a collection with *ELEMENT COUNT* elements (where key and value types are u64).
The red black is also persistent and implemented in rbtree.rs
(based on [Matt Might's article](http://matt.might.net/articles/red-black-delete/)).

| ELEMENT COUNT | HAMT | REDBLACK TREE | HASHMAP |
|:--------------|:----:|:-------------:|:-------:|
| 10            | 43   | 14            | 41      |
| 1000          | 59   | 64            | 45      |
| 100000        | 85   | 308           | 70      |

In percent over std::HashMap (less than 100% means faster, more means slower than std::HashMap).

| ELEMENT COUNT | HAMT | REDBLACK TREE | HASHMAP |
|:--------------|:----:|:-------------:|:-------:|
| 10            | 106% | 34%           | 100%    |
| 1000          | 130% | 140%          | 100%    |
| 100000        | 121% | 437%          | 100%    |

Both persistent implementations are quite fast but don't scale as well as the std::HashMap.
The HAMT is in the same ballpark as the std::HashMap, even for larger collections.
Both the HAMT and the regular HashMap still suffer a bit from Rust's currently slow hash
function. Otherwise, I guess they would be closer to the red-black tree for small collections.
~~Also, LLVM unfortunately does not (yet) properly translate the `cntpop` intrinsic function
(which could be just one CPU instruction on many architectures, but is translated to a much more
expensive instruction sequence currently).~~ As pointed out [on reddit](http://www.reddit.com/r/rust/comments/1xa8uy/a_persistent_map_implementation_like_in_clojure/cf9xm3a), properly configuring LLVM
(e.g. by setting the target-cpu option) is necessary for it to issue the popcnt instruction.

### Insertion
Times (in microseconds) for one thousand insertions into a collection with *ELEMENT COUNT* elements (again, key and value type is u64).

| ELEMENT COUNT | HAMT | REDBLACK TREE | HASHMAP |
|:--------------|:----:|:-------------:|:-------:|
| 10            | 186  | 1082          | 49      |
| 1000          | 252  | 1284          | 60      |
| 100000        | 1616 | 2756          | 76      |

In percent over std::HashMap (less than 100% means faster, more means slower than std::HashMap).

| ELEMENT COUNT | HAMT  | REDBLACK TREE | HASHMAP |
|:--------------|:-----:|:-------------:|:-------:|
| 10            | 377%  | 2198%         | 100%    |
| 1000          | 417%  | 2127%         | 100%    |
| 100000        | 2138% | 3646%         | 100%    |

As can be seen, the HAMT holds up pretty well against the non-persistent std::HashMap.

In conclusion, even with (atomic, multithreaded) refcounting a HAMT can perform pretty well :)
