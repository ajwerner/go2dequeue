Exercise building a dequeue with the go2 Type Parameter Draft
=======


This project is an exploration with the `go2go` / [Type Parameters - Design
Draft] with an eye towards block-based data structures which promote access
locality. 

Such data structures traditionally utilize blocks of objects laid out
contiguously in memory. In go, today, one generaly specializes such data
structures which are performance critical. This is especially important in the
case of reducing allocations which occur on critical paths. Using a sync.Pool
can help with allocations in some cases but can be clunky to use and, well,
do create more pointers for GC to worry about and don't have the access 
locality.

Whether it's really important to have data structures which include arrays of
other data structures laid out contiguously in RAM is really important is sort
of orthogonal. Let's just assume that it is for now and look at how we'd do it.
We only need to consider rejecting the importance of this case if we can't find
a good solution or idiom to support it. The [google/btree] library README links
this [Google Open Source Blog Post on block-based container] performance
indicating it does matter. The interesting thing about that library is it
hardly gets the access locality of the C++ library in the blog post it
references. 

The challenge this library explores is to layout blocks in structures
contiguously in RAM while allowing the user to have some control over that
block size.

## This example

The approach this experiment takes is to allow the users to specify the block
size by providing a function to map an array type to a slice. The allows the
blocks to contain a slice which references the array. The overhead to maintain
the slice header is 3 words but the cost is probably less in memory and more in
the optimizations the compiler will be less able to perform. In particular, it
probably will need to do bounds checking on that slice and there are probably
some opportunities to avoid the pointer interactions. The interface ends up
being pretty reasonable though a little bit ... opaque?

What this looked like in this demo is*

```go

// Say we want to create a dequeue of things.
type Thing struct { Foo int64, Bar int64, ID [16]byte }
// The below incantation will create one with blocks of 37 elements.
d := NewWithArray(func(a *[37]Thing) []Thing { return (*a)[:] })
// The nodes in the linked list of this structure are 1240 bytes, not that
// that's a great number or anything like that, just saying it's pretty big.
// Maybe there'd be something good to get out of aligning things are cache line
// sizes. See TestNodeSize.
```

## Other things you could do

### Use a slice for the buffer

A different approach would be to just have the array in the node, take a
buffer size and then keep a sync.Pool or freelist of slices. Today's
github.com/google/btree does this but way worse in that it just keeps a slice
of btree.Item. So fine, it's be two objects, the nodes and their item buffers.
Maybe that is the answer but it's pretty lame. My feeling is once somebody is
optimizing a data structure for access locality they are trying to optimize as
far as they can go.

### Ignore or augment the language generics with manual or automatic specialization

Today, in performance critical cases, people use a variety of automatic or
manual specialization. There are tools like https://github.com/cheekybits/genny
or https://github.com/mmatczuk/go_generics. Note that the latter has been used
to good effect in CockroachDB for building specialized copy-on-write, interval
B-trees.

One answer to this problem might just be that go's generics solution doesn't
solve for the problem of specializing block-based data structures. That's
not a particularly satisfying answer.

## What I might have expected / guess I'm proposing

Type lists are a thing for utilizing language features which only apply to
certain kinds of types. There should be a a way to create a type list for
arrays of types.

As a first pass I would have anticipated that there would be a way to inform
this package of a buffer size maybe drawn from a fixed number of sizes.
For example:

```go

type BlockSize int

const (
    _ BlockSize = 1 << (4*iota)
    Block16
    Block256
)

// Array here allows
type Array(type T) interface {
    type [Block16]T, [Block256]T
}

func New(type T)(blockSize BlockSize) Dequeue(T) {
    switch blockSize {
    case Block16: 
        return &(dequeue(T, [Block16]T){ ... })
    case Block256:
        return &(dequeue(T, [Block16]T){})
    }
}

type dequeue(type T interface{}, ArrayT Array(T)) struct {
    len
}

type node(type T interface{}, ArrayT Array(T)) struct {
    left, right *node(T, ArrayT)
    rb ringBuf(T, ArrayT)
}

type ringBuf(type T interface{}, ArrayT Array(T)) struct {
    head, len int
    // buf is known to be an array type with a size of either 16 or 256 so
    // I'd expect to be able to interact with this like an array. I don't think
    // that that works today.
    buf ArrayT 
}
```

The above also seems needlessly clunky. I'd prefer if I could just represent
all array types like this:

```go
type Array(type T) interface {
    [...]T
}
```

Then I could do the following and everything else should just fall out.

```go
func New(type T interface{}, ArrayT Array(T))() Dequeue(T) {
    return &(dequeue(T, ArrayT){})
}
```

The above feels to me to be compatible with the goals and idioms of the proposal
but I could be missing out on some major complexity. Hope y'all find this
helpful.

## Aside

It would be cool to be able to embed a generic type in another generics type.
I can see why it's annoying because the name of the field will get totally
mangled in the specialization but maybe that's not so bad? Maybe you'd actually
call the embedded type by its unspecialized type name even in the specialized
version. 

I would have liked to embed the `ringBuf` in the `node`. Given it's not
embedded it's almost not worth abstracting. 

[Type Parameters - Design Draft]: https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md
[Google Open Source Blog Post on block-based container]: https://opensource.googleblog.com/2013/01/c-containers-that-save-memory-and-time.html
[google/btree]: https://github.com/google/btree

