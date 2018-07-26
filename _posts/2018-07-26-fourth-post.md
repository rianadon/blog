---
layout: post
title:  "Emoji vectors and optimization. Oh my! (Part 3: Assembly and assembly)"
date:   2018-07-26
---

> This is Part 3 - the third and final part. Because it's 3rd, that means there were two posts before it. These detailed how I'm using dot products to compare vectors that corresponded to words corresponding to emoji. Unless you came here for only the asm.js, those posts give some nice background info. Also, if you were expecting proof that asm.js solves every problem out there, this is not the post for you. As a warning, things go badly.

JavaScript can be inefficient. Taking dot products over 2800 length-300 vectors is a lengthy process. I ran this computation from my last blog post 100 times in Chrome:

```javascript
function dotProduct(view1, index1, view2, index2) {
    let dot = 0
    for (let i = 0; i < 300; i++) { // Each word vector is length 300
        dot += view1.getFloat32(index1 * 300 * 4 + i * 4, true)
             * view2.getFloat32(index2 * 300 * 4 + i * 4, true)
    }
    return dot
}

// The part below gets put in a for loop to run 100 times

const products = emojiVocab.map((_, emojiIndex) => {
    return [emojiIndex, dotProduct(weights, index, emojiWeights, emojiIndex)]
})
```

and got the following results:

```
On phone:  44277.39999999176 ms / 100 = 442.7739999999176 ms
On laptop: 13675.79999999725 ms / 100 = 136.7579999999725 ms
```

While 0.1 seconds isn't bad, almost half a second is noticeable. So what's the problem here? Well, each dot product multiplies 300 floating point numbers and then adds them all together. Let's call that 600 floating point operations. Then multiply that by the 2800 vectors being multiplied and you get 1.68 megaFLOPs. That's a lot of floating point operations. On my phone, this comes out to a speed of 3.8 megaFLOPS. Now most processors support *at least* one flop per instruction cycle. Then multiply that by maybe 4 for the extra indexing operations, and you get 15.2 Mhz. This means my Snapdragon 810 phone is running JavaScript slower than an Arduino Uno runs (16 Mhz).

How can this be fixed? Let's drop JavaScript and move to assembly!

## Handwriting Asm.js

There's a thing called emscripten out there. You can compile a little C or C++ program that prints "hello world" into 50 Kb of JavaScript. Yes, it may necessary boilerplate, but I need to implement about 10 lines of JavaScript here. 1% code / 99% boilerplate is not a good ratio. So I ignore popular recommendations and write some asm.js myself. It's actually not too terrible.

### Asm.js basics

The very first asm.js example I found that works was [this little Gist](https://gist.github.com/yomotsu/4ff8a3f441d16fd81250). For some reason Chrome complained about the syntax when running examples taken from the [asm.js specification](http://asmjs.org/spec/latest/).

So let's start with the (almost) simplest asm.js code I can get:

```javascript
function AsmModule(stdlib, foreign, buffer) {
    'use asm'

    function test() {
        return 0|0
    }

    return { test: test };
}

console.log(AsmModule().test())
```

What's going on here?

First, asm.js likes modules. The standard way of running asm.js code is that you define an Asm module, pass it a few parameters, then use it's exported methods (in here only `test`).

These parameters allow you to reference `Math` modules, use JavaScript methods, and use a heap. To quote the specification:

> An asm.js module can take up to three optional parameters, providing access to external JavaScript code and data:
>
>* a **standard library** object, providing access to a limited subset of the JavaScript [standard libraries](http://asmjs.org/spec/latest/#standard-library);
> * a **foreign function interface** (FFI), providing access to custom external JavaScript functions; and
> * a **heap buffer**, providing a single [`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/JavaScript/Typed_arrays/ArrayBuffer) to act as the asm.js heap.

For now we don't need any. The standard library and heap buffer will come in use later.

You'll also notice (maybe) the weird `0|0`. No, this is not some kind of Unicode art owl eyes thing (`ಠ|ಠ`). This is how you specify types. There's a really nice reference for this kind of stuff [here](https://danigb.github.io/2017/asmjs). Anyways, the `|0` makes the `0` return value an integer.

### A little more complicated...

Since the code will eventually have to compute dot products, let's start out with a little multiplication (with single-precision floats since that's what the code used before).

Sounds easy right? Let's take a stab at it:

```javascript
function AsmModule(stdlib, foreign, buffer) {
    'use asm'

    function test(a, b) {
        return a * b
    }

    return { test: test };
}

console.log(AsmModule().test(0.4, 4.0))
```

That should give `1.6` right?

Unless `Invalid asm.js: Unexpected token` equals `1.6` (this *is* JavaScript, so who knows), there's an oopsie.

Asm.js is typed. Here there are no types. `a` could be an integer, double, or float. the compiler has no idea. Here, it's expecting type declarations for the parameters. That's done by adding type annotations as such:

```javascript
var fround = stdlib.Math.fround

function test(a, b) {
    a = fround(a)
    b = fround(b)

    return a * b
}
```

First, `fround` gets taken from the JavaScript's math library. This is how asm.js declares things as floats. The function normally rounds floats tot he nearest single precision float, so that's a good choice for making the script run the same with and without asm!

Now the module requires this magic `stdlib`. We can just pass `window` for that:

```javascript
console.log(AsmModule(window).test(0.4, 4.0))
```

Now when running the code, Chrome gives a `Invalid asm.js: Invalid return type`. That's a little better! All that has to be done now is to also use `fround` to document the return type:

```javascript
return fround(a * b)
```

And that gives `1.600000023841858` as the answer. Wallah!

### Hip-hoppity-heap

All the numbers we want to dot-product-ize are in an `ArrayBuffer`. That'll have to eventually be passed to asm.js.

To pass a heap, it first has to be created:

```javascript
var heap = new ArrayBuffer(0x10000)
```

Here, `0x10000` is the minimum size the heap can have.

The heap then gets passed as the third parameter to the module:

```javascript
console.log(AsmModule(window, null, heap).test(0, 4))
```

Since the floats are going to be stored in the heap, I've replaced the `0.4` and `4.0` (the floats to multiply), with their indexes in the ArrayBuffer. Single precision floats take up 4 bytes, so the indexes are offset by 4.

Speaking of storing the floats in the heaps, we can do that with a [`DataView`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView):

```javascript
var view = new DataView(heap)
view.setFloat32(0, 0.4, true)
view.setFloat32(4, 4.0, true)
```

I'm going to be assuming my system is little endian. If the system isn't, it isn't going to work with the little-endian data I'm going to give it. We'll have to hope for the best.

Then to write the asm function:

```javascript
var heap = new stdlib.Float32Array(buffer) // Cast the heap to a bunch of 32 bit floats

function test(a, b) {
    // Declare a and b to be integers
    a = a|0
    b = b|0

    return fround(heap[a>>z] * heap[b>>2])
}
```


The `>>2` is needed to convert the byte indexes passed in to the float indexes of the `Float32Array`. Why couldn't we have just passed in the float indexes like `test(0, 1)`? The asm.js compiler throws the error `Invalid asm.js: Expected shift of word size`. Who knows why that's required.

Anyways, that gives back `1.600000023841858` as the result, same as before. Nothing new.

### Dot products

Now that we've got multiplication down let's try addition (that's a great order isn't it?).

We'll take the dot product of two length-300 vectors. The code sans explanation is below as I've already spent two sections explaining things:

```javascript
function AsmModule(stdlib, foreign, buffer) {
    'use asm'

    var fround = stdlib.Math.fround
    var heap = new stdlib.Float32Array(buffer)

    var vectorlength = 300 // In declarations we don't need the "|"

    function dotprod(a, b) {
        // These parameters will be reused as indexes
        a = a|0
        b = b|0

        var prod = fround(0.0), max = 0

        // Asm.js likes computations verbosely typed
        max = (a + vectorlength<<2)|0; // shift vector length out by 2
                                       // to account for byte indexing
        while ((a|0) < (max|0)) { // This loop should run 300 times
            prod = fround(prod + fround(heap[a>>2] * heap[b>>2]))
            // increment the indexes
            a = (a + 4)|0
            b = (b + 4)|0
        }

        return prod // the type of prod was already defined so no fround needed here
    }

    return { dotprod: dotprod };
}

const heap = new ArrayBuffer(0x10000)
const view = new DataView(heap)
// Take the dot product of the vectors <1, 2, 3, ... 10, 0, 0 ...>
// and <1, 2, 3, ... 10, 0, 0 ...> = sum of squares up to and including 10
for (let i = 1; i <= 10; i++) {
    view.setFloat32(i<<2, i, true)
    view.setFloat32(300+i<<2, i, true)
}
console.log(AsmModule(window, null, heap).dotprod(0, 300*4))
```

Now I just need to do this 2800 times.

I'll organize the heap in the following manner.

## Incorporating the change

Before I had a `dotProduct` function that given two arrays and two indexes, would return the dot product of the two 300-length vectors at those indexes. Below is a rewrite of a function to return the function defined above. This new `createdotter` function allows the asm module to be initialized only once.

```javascript
function createdotter(emojiWeights) {
    // Create the heap
    const heapUnit = 0x100000 // heap size must be rounded to this
    const heapSize = emojiWeights.byteLength + 300*4 // desired size
    const heap = new ArrayBuffer(Math.ceil(heapSize / heapUnit) * heapUnit)

    // Populate the heap with the emoji weights
    const heapArray = new Uint8Array(heap)
    const weightsArray = new Uint8Array(emojiWeights.buffer, emojiWeights.byteOffset)
    heapArray.set(weightsArray, 300*4)

    // Set up the module
    const mod = AsmModule(window, null, heap)


    return function dotProduct(view1, index1, _, emojiIndex) {
        // Populate the heap with the vector to find the dot product with
        const viewArray = new Uint8Array(view1.buffer, view1.byteOffset)
        const slice = viewArray.slice(index1*300*4, (index1+1)*300*4)
        heapArray.set(slice, 0)

        // Call the asm.js module
        return mod.dotprod(0, (emojiIndex+1)*300*4)
    }
}
```

With this change I get the following new performance numbers for the code at the beginning of the post (and using the new dotProduct function):

```
On phone:  1735.899999999674 ms / 100 = 17.35899999999674 ms
On laptop:  745.900000008987 ms / 100 =  7.45900000008987 ms
```

Old timings for reference:

```
On phone:  44277.39999999176 ms / 100 = 442.7739999999176 ms
On laptop: 13675.79999999725 ms / 100 = 136.7579999999725 ms
```

That's much faster!

# TL;DR `DataView` is slow

For fun (i.e. because I had some errors and the asm.js wasn't compiling), I tried removing the `'use asm'`, so all code ran in normal JavaScript. Interestingly, the results weren't too much worse:

```
On phone:  2094.8000000062166 ms / 100 = 20.948000000062166 ms
On laptop: 1315.3999999922235 ms / 100 = 13.153999999922235 ms
```

The only major difference between the asm.js code and the old code is that the former uses `Float32Array`s and the latter uses `DataView`s. So I swapped out the APIs for each other in my code (modifying the old `dotProduct` function), and got the following results:

```
On phone:  852.1999999938998 ms / 100 = 8.521999999938998 ms
On laptop: 374.0999999863561 ms / 100 = 3.740999999863561 ms
```

So using asm.js actually made things *slower* than when using `Float32Array`s. That's likely the result of the copying of the compare vector into the heap. Oh well, that just goes to show how optimized JavaScript is on its own, without extensions/limits such as `asm.js`.

Unfortunately there's no part 4. I'm sorry. There's only the [online implementation of this stuff](https://rianadon.github.io/Emoji-Suggester-v2/).
