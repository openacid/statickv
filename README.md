# succinct

[![Travis](https://travis-ci.com/openacid/succinct.svg?branch=main)](https://travis-ci.com/openacid/succinct)
![test](https://github.com/openacid/succinct/workflows/test/badge.svg)

[![Report card](https://goreportcard.com/badge/github.com/openacid/succinct)](https://goreportcard.com/report/github.com/openacid/succinct)
[![Coverage Status](https://coveralls.io/repos/github/openacid/succinct/badge.svg?branch=main&service=github)](https://coveralls.io/github/openacid/succinct?branch=main&service=github)

[![GoDoc](https://godoc.org/github.com/openacid/succinct?status.svg)](http://godoc.org/github.com/openacid/succinct)
[![PkgGoDev](https://pkg.go.dev/badge/github.com/openacid/succinct)](https://pkg.go.dev/github.com/openacid/succinct)
[![Sourcegraph](https://sourcegraph.com/github.com/openacid/succinct/-/badge.svg)](https://sourcegraph.com/github.com/openacid/succinct?badge)

succinct provides several static succinct data types

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Succinct Set](#succinct-set)
  - [Synopsis](#synopsis)
  - [Performance](#performance)
  - [Implementation](#implementation)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Succinct Set

中文介绍: [100行代码的压缩前缀树: 50% smaller](https://blog.openacid.com/algo/succinctset/)

Set is a succinct, sorted and static string set impl with compacted trie as
storage. The space cost is about half lower than the original data.

## Synopsis

```go
package succinct

import "fmt"

func ExampleNewSet() {
	keys := []string{
		"A", "Aani", "Aaron", "Aaronic", "Aaronical", "Aaronite",
		"Aaronitic", "Aaru", "Ab", "Ababdeh", "Ababua", "Abadite",
	}
	s := NewSet(keys)
	for _, k := range []string{"Aani", "Foo", "Ababdeh"} {
		found := s.Has(k)
		fmt.Printf("lookup %10s, found: %v\n", k, found)
	}

	// Output:
	//
	// lookup       Aani, found: true
	// lookup        Foo, found: false
	// lookup    Ababdeh, found: true
}
```

## Performance

-   200 kilo real-world words collected from web:
    - the space costs is **57%** of original data size.
    - And a `Has()` costs about `350 ns` with a **zip-f** workload.

    Original size: 2204 KB

    With comparison with string array bsearch and google [btree][] :

    | Data         | Engine       | Size(KB) | Size/original | ns/op |
    | :--          | :--          | --:      | --:           | --:   |
    | 200kweb2     | bsearch      |  5890    |  267%         | 229   |
    | 200kweb2     | succinct.Set |  1258    |   57%         | 356   |
    | 200kweb2     | btree        | 12191    |  553%         | 483   |

    > A string in go has two fields: a pointer to the text content and a length.
    > Thus the space overhead is quite high with small strings.
    > [btree][] internally has more pointers and indirections(interface).


-   870 kilo real-world ipv4:
    - the space costs is **67%** of original data size.
    - And a `Has()` costs about `500 ns` with a **zip-f** workload.

    Original size: 6823 KB

    | Data         | Engine       | Size(KB) | Size/original | ns/op |
    | :--          | :--          | --:      | --:           | --:   |
    | 870k_ip4_hex | bsearch      | 17057    |  500%         | 276   |
    | 870k_ip4_hex | succinct.Set |  2316    |   67%         | 496   |
    | 870k_ip4_hex | btree        | 40388    | 1183%         | 577   |



## Implementation

It stores sorted strings in a compacted trie(AKA prefix tree). A trie node has
at most 256 outgoing labels. A label is just a single byte. E.g., [ab, abc,
abcd, axy, buv] is represented with a trie like the following: (Numbers are node
id)

    ^ -a-> 1 -b-> 3 $
      |      |      `c-> 6 $
      |      |             `d-> 9 $
      |      `x-> 4 -y-> 7 $
      `b-> 2 -u-> 5 -v-> 8 $

Internally it uses a packed []byte and a bitmap with `len([]byte)` bits to
describe the outgoing labels of a node,:

    ^: ab  00
    1: bx  00
    2: u   0
    3: c   0
    4: y   0
    5: v   0
    6: d   0
    7: ø
    8: ø
    9: ø

In storage it packs labels together and bitmaps joined with separator `1`:

    labels(ignore space): "ab bx u c y v d"
    label bitmap:          0010010101010101111

In this way every node has a `0` pointing to it(except the root node)
and has a corresponding `1` for it:

                                   .-----.
                           .--.    | .---|-.
                           |.-|--. | | .-|-|.
                           || ↓  ↓ | | | ↓ ↓↓
    labels(ignore space):  ab bx u c y v d øøø
    label bitmap:          0010010101010101111
    node-id:               0  1  2 3 4 5 6 789
                              || | ↑ ↑ ↑ |   ↑
                              || `-|-|-' `---'
                              |`---|-'
                              `----'

To walk from a parent node along a label to a child node, count the number of
`0` upto the bit the label position, then find where the the corresponding
`1` is:

    childNodeId = select1(rank0(i))

In our impl, it is:

    nodeId = countZeros(ss.labelBitmap, ss.ranks, bmIdx+1)
    bmIdx = selectIthOne(ss.labelBitmap, ss.ranks, ss.selects, nodeId-1) + 1

Finally leaf nodes are indicated by another bitmap `leaves`, in which a `1` at
i-th bit indicates the i-th node is a leaf:

    leaves: 0001001111

# License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

[btree]: https://github.com/google/btree