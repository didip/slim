<!--
based on the a great readme template
https://gist.github.com/PurpleBooth/109311bb0361f32d87a2
-->

# Slim - surprisingly space efficient data types in Golang

[![Travis](https://travis-ci.com/openacid/slim.svg?branch=main)](https://travis-ci.com/openacid/slim)
![test](https://github.com/openacid/slim/workflows/test/badge.svg)

[![Report card](https://goreportcard.com/badge/github.com/openacid/slim)](https://goreportcard.com/report/github.com/openacid/slim)
[![Coverage Status](https://coveralls.io/repos/github/openacid/slim/badge.svg?branch=main&service=github)](https://coveralls.io/github/openacid/slim?branch=main&service=github)

[![GoDoc](https://godoc.org/github.com/openacid/slim?status.svg)](http://godoc.org/github.com/openacid/slim)
[![PkgGoDev](https://pkg.go.dev/badge/github.com/openacid/slim)](https://pkg.go.dev/github.com/openacid/slim)
[![Sourcegraph](https://sourcegraph.com/github.com/openacid/slim/-/badge.svg)](https://sourcegraph.com/github.com/openacid/slim?badge)

Slim is collection of surprisingly space efficient data types, with
corresponding serialization APIs to persisting them on-disk or for transport.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Why slim](#why-slim)
- [Performance and memory overhead](#performance-and-memory-overhead)
- [Synopsis](#synopsis)
  - [1. Index on-disk key-values](#1-index-on-disk-key-values)
  - [2. Sparse index](#2-sparse-index)
  - [3. Range scan](#3-range-scan)
- [Filter mode and KV mode.](#filter-mode-and-kv-mode)
- [Try it](#try-it)
  - [Versions](#versions)
- [Who are using slim](#who-are-using-slim)
- [Feedback and contributions](#feedback-and-contributions)
- [Authors](#authors)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Why slim

As data on internet keeps increasing exponentially,
the capacity gap between memory and disk becomes greater.

Most of the time, a data itself does not need to be loaded into expensive main memory.
Only the much more important information, WHERE-A-DATA-IS, deserve a seat in
main memory.


This is what `slim` does, keeps as little information as possible in main
memory, as a minimized index of huge amount external data.

-   `SlimIndex`: is a common index structure, building on top of `SlimTrie`.

    [![GoDoc](https://godoc.org/github.com/openacid/slim/index?status.svg)](http://godoc.org/github.com/openacid/slim/index)

-   `SlimTrie` is the underlying index data structure, evolved from [trie][].

    [![GoDoc](https://godoc.org/github.com/openacid/slim/trie?status.svg)](http://godoc.org/github.com/openacid/slim/trie)

    **Features**:

    -   **Minimized**:
        **11 bits per key**(far less than an 64-bits pointer!!).

    -   **Stable**:
        memory consumption is stable in various scenarios.
        The Worst case converges to average consumption tightly.
        See benchmark.

    -   **Loooong keys**:
        You can have **VERY** long keys(`16K bytes`), without any waste of memory(and money).
        Do not waste your life writing another prefix compression`:)`.
        ([aws-s3][] limits key length to 1024 bytes).
        Memory consumption only relates to key count, **not to key length**.

    -   **Ordered**:
        like [btree][], keys are stored.
        Range-scan will be ready in `0.6.0`.

    -   **Fast**:
        **~150 ns** per `Get()`.
        Time complexity for a get is `O(log(n) + k); n: key count; k: key length`.

    -   **Ready for transport**:
        a single `proto.Marshal()` is all it requires to serialize, transport or persisting on disk etc.


## Performance and memory overhead

- **3.3 times faster** than the [btree][].
- **2.3 times faster** than binary search.

![](trie/report/bench_msab_present_zipf.jpg)

- **Memory overhead is about 11 bit per key**.

![](trie/report/mem_usage.jpg)


Read more about [Performance](docs/performance.md)



## Synopsis

### 1. Index on-disk key-values

One of the typical usages of slim is to index serialized data on disk(e.g., key value records in a SSTable).
By keeping a slim in memory, one can quickly find the on-disk offset of the record by a key.

<details>
    <summary>Show me the code ......</summary>

```go
package index_test

import (
	"fmt"
	"strings"

	"github.com/openacid/slim/index"
)

type Data string

func (d Data) Read(offset int64, key string) (string, bool) {
	kv := strings.Split(string(d)[offset:], ",")[0:2]
	if kv[0] == key {
		return kv[1], true
	}
	return "", false
}

func Example() {

	// Accelerate external data accessing (in memory or on disk) by indexing
	// them with a SlimTrie:

	// `data` is a sample of some unindexed data. In our example it is a comma
	// separated key value series.
	//
	// In order to let SlimTrie be able to read data, `data` should have
	// a `Read` method:
	//     Read(offset int64, key string) (string, bool)
	data := Data("Aaron,1,Agatha,1,Al,2,Albert,3,Alexander,5,Alison,8")

	// keyOffsets is a prebuilt index that stores key and its offset in data accordingly.
	keyOffsets := []index.OffsetIndexItem{
		{Key: "Aaron", Offset: 0},
		{Key: "Agatha", Offset: 8},
		{Key: "Al", Offset: 17},
		{Key: "Albert", Offset: 22},
		{Key: "Alexander", Offset: 31},
		{Key: "Alison", Offset: 43},
	}

	// `SlimIndex` is simply a container of SlimTrie and its data.
	st, err := index.NewSlimIndex(keyOffsets, data)
	if err != nil {
		fmt.Println(err)
	}

	// Lookup
	v, found := st.Get("Alison")
	fmt.Printf("key: %q\n  found: %t\n  value: %q\n", "Alison", found, v)

	v, found = st.Get("foo")
	fmt.Printf("key: %q\n  found: %t\n  value: %q\n", "foo", found, v)

	// Output:
	// key: "Alison"
	//   found: true
	//   value: "8"
	// key: "foo"
	//   found: false
	//   value: ""
}
```

</details>

### 2. Sparse index

Create an index item for every 4(or more as you wish) keys.

Let several adjacent keys share one index item reduces a lot memory
cost if there are huge amount keys in external data.
Such as to index billions of 4KB objects on a 4TB disk(because one disk IO
costs 20ms for either reading 4KB or reading 1MB).

<details>
    <summary>Show me the code ......</summary>

```go
package index_test

import (
	"fmt"
	"strings"

	"github.com/openacid/slim/index"
)

type RangeData string

func (d RangeData) Read(offset int64, key string) (string, bool) {
	for i := 0; i < 4; i++ {
		if int(offset) >= len(d) {
			break
		}

		kv := strings.Split(string(d)[offset:], ",")[0:2]
		if kv[0] == key {
			return kv[1], true
		}
		offset += int64(len(kv[0]) + len(kv[1]) + 2)

	}
	return "", false
}

func Example_indexRanges() {

	// Index ranges instead of keys:
	// In this example at most 4 keys shares one index item.

	data := RangeData("Aaron,1,Agatha,1,Al,2,Albert,3,Alexander,5,Alison,8")

	// keyOffsets is a prebuilt index that stores range start, range end and its offset.
	keyOffsets := []index.OffsetIndexItem{
		// Aaron  +--> 0
		// Agatha |
		// Al     |
		// Albert |

		// Alexander +--> 31
		// Alison    |

		{Key: "Aaron", Offset: 0},
		{Key: "Agatha", Offset: 0},
		{Key: "Al", Offset: 0},
		{Key: "Albert", Offset: 0},

		{Key: "Alexander", Offset: 31},
		{Key: "Alison", Offset: 31},
	}

	st, err := index.NewSlimIndex(keyOffsets, data)
	if err != nil {
		panic(err)
	}

	v, found := st.RangeGet("Aaron")
	fmt.Printf("key: %q\n  found: %t\n  value: %q\n", "Aaron", found, v)

	v, found = st.RangeGet("Al")
	fmt.Printf("key: %q\n  found: %t\n  value: %q\n", "Al", found, v)

	v, found = st.RangeGet("foo")
	fmt.Printf("key: %q\n  found: %t\n  value: %q\n", "foo", found, v)

	// Output:
	// key: "Aaron"
	//   found: true
	//   value: "1"
	// key: "Al"
	//   found: true
	//   value: "2"
	// key: "foo"
	//   found: false
	//   value: ""
}
```

</details>


### 3. Range scan

Slim can also be used as a traditional in-memory kv-store:
Building a slim with `Opt{ Complete: Bool(true) }`,
it won't strip out any information(e.g., it won't eliminate single-branch labels)
and it will functions the same as a `btree`.
This snippet shows how to iterate key values.

<details>
    <summary>Show me the code ......</summary>

```go
package trie

import (
	"fmt"

	"github.com/openacid/slim/encode"
)

func ExampleSlimTrie_ScanFrom() {
	var keys = []string{
		"",
		"`",
		"a",
		"ab",
		"abc",
		"abca",
		"abcd",
		"abcd1",
		"abce",
		"be",
		"c",
		"cde0",
		"d",
	}
	values := makeI32s(len(keys))

	codec := encode.I32{}
	st, _ := NewSlimTrie(codec, keys, values, Opt{
		Complete: Bool(true),
	})

	// untilD stops when encountering "d".
	untilD := func(k, v []byte) bool {
		if string(k) == "d" {
			return false
		}

		_, i32 := codec.Decode(v)
		fmt.Println(string(k), i32)
		return true
	}

	fmt.Println("scan (ab, +∞):")
	st.ScanFrom("ab", false, true, untilD)

	fmt.Println()
	fmt.Println("scan [be, +∞):")
	st.ScanFrom("be", true, true, untilD)

	fmt.Println()
	fmt.Println("scan (ab, be):")
	st.ScanFromTo(
		"ab", false,
		"be", false,
		true, untilD)

	// Output:
	//
	// scan (ab, +∞):
	// abc 4
	// abca 5
	// abcd 6
	// abcd1 7
	// abce 8
	// be 9
	// c 10
	// cde0 11
	//
	// scan [be, +∞):
	// be 9
	// c 10
	// cde0 11
	//
	// scan (ab, be):
	// abc 4
	// abca 5
	// abcd 6
	// abcd1 7
	// abce 8
}
```

</details>


## Filter mode and KV mode.

Slim can be built into either a filter(like `bloom filter` but with key order preserved.) or a real kv-store(like `btree`)
There is an `option` in `NewSlimTrie(..., option)` to control the building behavior.
Ref: [Opt](https://pkg.go.dev/github.com/openacid/slim@v0.5.11/trie#Opt)

- To use slim as a kv-store, set the option to `Complete` then there won't be false positives.

- To use it as a filter, set `InnerPrefix`, `LeafPrefix` to false(`Complete` implies `InnerPrefix==true` and `LeafPrefix==true`).
  Then slim won't store any single branch label in the trie it builds.

  With  `InnerPrefix==true`, it does not reduce a single label branch that leads to an inner node.

  With  `LeafPrefix==true`, it does not reduce a single label branch that leads to a leaf node.

  E.g.:

  ```
  // Complete
  InnerPrefix: true
  LeafPrefix: true
  ^ -a-> 1 -b-> $
   `-c-> 2 -x-> 3 -y-> $
                 `-z-> $
  
  InnerPrefix: true
  LeafPrefix: false
  ^ -a-> $
   `-c-> 2 -x-> 3 -y-> $
                 `-z-> $
  
  InnerPrefix: false
  LeafPrefix: true
  ^ -a-> 1 -b-> $
   `-c-> 3 -y-> $
          `-z-> $
  
  InnerPrefix: false
  LeafPrefix: false
  ^ -a-> $
   `-c-> 3 -y-> $
          `-z-> $
  ```

The memory consumption in filter mode and kv mode differs significantly.
The following chart shows memory consumption by 1 million var-length string, 10 to 20 byte in different mode:

| -                | size  | gzip-size |
| :--              | --:   | --:       |
| sample data size | 15.0M | 14.0M     |
| Complete:true    | 14.0M | 10.0M     |
| InnerPrefix:ture |  1.3M |  0.9M     |
| all false        |  1.3M |  0.8M     |


<!-- ## FAQ -->

## Try it

**Install**

```sh
go get github.com/openacid/slim/trie
```

Change-log: [Change-log](docs/change-log.yaml)

### Versions

A newer version `y` being compatible with an older version `x` means `y` can
load data serialized by `x`. But `x` should never try to load data serialized by
a newer version `y`.

- `v0.5.*` is compatible with `0.2.*`, `0.3.*`, `0.4.*`, `0.5.*`.
- `v0.4.*` is compatible with `0.2.*`, `0.3.*`, `0.4.*`.
- `v0.3.*` is compatible with `0.2.*`, `0.3.*`.
- `v0.2.*` is compatible with `0.2.*`.


<!-- TODO add FAQ -->
<!-- TODO add serialization explanation, on-disk data structure etc. -->

## Who are using slim

<span> <span> ![][baishancloud-favicon] </span> <span> [baishancloud][] </span> </span>

<!-- ## Slim internal -->

<!-- ### Built With -->

<!-- - [protobuf][] - Define on-disk data-structure and serialization engine. -->
<!-- - [dep][] - Dependency Management. -->
<!-- - [semver][] - For versioning data-structure. -->

<!-- ### Directory Layout -->

<!-- We follow the: [golang-standards-project-layout][]. -->

<!-- [> TODO read the doc and add more standards <] -->

<!-- -   `vendor/`: dependency packages. -->
<!-- -   `prototype/`: on-disk data-structure. -->
<!-- -   `docs/`: documents about design, trade-off, etc -->
<!-- -   `tools/`: documents about design, trade-off, etc -->
<!-- -   `expamples/`: documents about design, trade-off, etc -->

<!-- Other directories are sub-package. -->


<!-- ### Versioning -->

<!-- We use [SemVer](http://semver.org/) for versioning. -->

<!-- For the versions available, see the [tags on this repository](https://github.com/your/project/tags).  -->

<!-- ### Data structure explained -->
<!-- [> TODO  <] -->

<!-- ## Limitation -->
<!-- [> TODO  <] -->


<!-- -   [ ] bitrie: 1 byte-per-key implementation. -->
<!-- -   [ ] balanced bitrie: which gives better worst-case performance. -->
<!-- -   [ ] generalised API as a drop-in replacement for map etc. -->


## Feedback and contributions

**Feedback and Contributions are greatly appreciated**.

At this stage, the maintainers are most interested in feedback centered on:

-   Do you have a real life scenario that `slim` supports well, or doesn't support at all?
-   Do any of the APIs fulfill your needs well?

Let us know by filing an issue, describing what you did or wanted to do, what
you expected to happen, and what actually happened:

-   [bug-report][]
-   [improve-document][]
-   [feature-request][]

Or other type of [issue][new-issue].

<!-- ## Contributing -->
<!-- The maintainers actively manage the issues list, and try to highlight issues -->
<!-- suitable for newcomers. -->

<!-- [> TODO dep CONTRIBUTING <] -->
<!-- The project follows the typical GitHub pull request model. See CONTRIBUTING.md for more details. -->

<!-- Before starting any work, please either comment on an existing issue, -->
<!-- or file a new one. -->

<!-- [> TODO  <] -->
<!-- Please read [CONTRIBUTING.md][] -->
<!-- for details on our code of conduct, and the process for submitting pull requests to us. -->
<!-- https://gist.github.com/PurpleBooth/b24679402957c63ec426 -->


<!-- ### Code style -->

<!-- ### Tool chain -->

<!-- ### Customized install -->

<!-- Alternatively, if you have a customized go develop environment, you could also -->
<!-- clone it: -->

<!-- ```sh -->
<!-- git clone git@github.com:openacid/slim.git -->
<!-- ``` -->

<!-- As a final step you'd like have a test to see if everything goes well: -->

<!-- ```sh -->
<!-- cd path/to/slim/build/pseudo-gopath -->
<!-- export GOPATH=$(pwd) -->
<!-- go test github.com/openacid/slim/array -->
<!-- ``` -->

<!-- Another reason to have a `pseudo-gopath` in it is that some tool have their -->
<!-- own way conducting source code tree. -->
<!-- E.g. [git-worktree](https://git-scm.com/docs/git-worktree) -->
<!-- checkouts source code into another dir other than the GOPATH work space. -->

<!-- ## Update dependency -->

<!-- Dependencies are tracked by [dep](https://github.com/golang/dep). -->
<!-- All dependencies are kept in `vendor/` dir thus you do not need to do anything -->
<!-- to run it. -->

<!-- You need to update dependency only when you bring in new feature with other dependency. -->

<!-- -   Install `dep` -->

<!--     ``` -->
<!--     curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh -->
<!--     ``` -->

<!-- -   Download dependency -->

<!--     ``` -->
<!--     dep ensure -->
<!--     ``` -->

<!--     > dep uses Gopkg.toml Gopkg.lock to track dependency info. -->
<!--     >  -->
<!--     > Gopkg.toml Gopkg.lock is created with `dep init`. -->
<!--     > -->
<!--     > dep creates a `vendor` dir to have all dependency package there. -->

<!-- See more: [dep-install][] -->


## Authors

<!-- ordered by unicode of author's name -->
<!-- leave 3 to 5 major jobs you have done in this project -->

- ![][刘保海-img-sml] **[刘保海][]** *marshaling*
- ![][吴义谱-img-sml] **[吴义谱][]** *array*
- ![][张炎泼-img-sml] **[张炎泼][]** *slimtrie design*
- ![][李文博-img-sml] **[李文博][]** *trie-compressing, trie-search*
- ![][李树龙-img-sml] **[李树龙][]** *marshaling*


See also the list of [contributors][] who participated in this project.


## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

<!-- ## Acknowledgments -->

<!-- [> TODO  <] -->
<!-- - Hat tip to anyone whose code was used -->

<!-- - Inspiration -->
<!--     patricial tree -->
<!--     fusion tree -->
<!--     critic trie -->
<!-- - etc -->

<!-- links -->

<!-- Bio -->

[刘保海]: https://github.com/liubaohai
[吴义谱]: https://github.com/pengsven
[张炎泼]: https://github.com/drmingdrmer
[李文博]: https://github.com/wenbobuaa
[李树龙]: https://github.com/lishulong

<!-- avatar -->

[刘保海-img-sml]: https://avatars1.githubusercontent.com/u/26271283?s=36&v=4
[吴义谱-img-sml]: https://avatars3.githubusercontent.com/u/6927668?s=36&v=4
[张炎泼-img-sml]: https://avatars3.githubusercontent.com/u/44069?s=36&v=4
[李文博-img-sml]: https://avatars1.githubusercontent.com/u/11748387?s=36&v=4
[李树龙-img-sml]: https://avatars2.githubusercontent.com/u/13903162?s=36&v=4

[contributors]: https://github.com/openacid/slim/contributors

[dep]: https://github.com/golang/dep
[protobuf]: https://github.com/protocolbuffers/protobuf
[semver]: http://semver.org/

[protoc-install]: http://google.github.io/proto-lens/installing-protoc.html
[dep-install]: https://github.com/golang/dep#installation

[CONTRIBUTING.md]: CONTRIBUTING.md

[baishancloud]: http://www.baishancdnx.com
[baishancloud-favicon]: http://www.baishancdnx.com/public/favicon.ico
[golang-standards-project-layout]: https://github.com/golang-standards/project-layout

<!-- issue links -->

[bug-report]:       https://github.com/openacid/slim/issues/new?labels=bug&template=bug_report.md
[improve-document]: https://github.com/openacid/slim/issues/new?labels=doc&template=doc_improve.md
[feature-request]:  https://github.com/openacid/slim/issues/new?labels=feature&template=feature_request.md

[new-issue]: https://github.com/openacid/slim/issues/new/choose

<!-- benchmark -->

[benchmark-get-png]: docs/trie/charts/bench_get_2019_06_03.png

<!-- links to other resource -->

<!-- reference -->

[trie]: https://en.wikipedia.org/wiki/Trie
[btree]: https://github.com/google/btree
[aws-s3]: https://aws.amazon.com/s3/
[red-black-tree]: https://en.wikipedia.org/wiki/Red%E2%80%93black_tree