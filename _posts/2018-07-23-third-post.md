---
layout: post
title:  "Emoji vectors and optimization. Oh my! (Part 2: Npy files in the browser)"
date:   2018-07-23
---

In Part 1, I discussed how to implement a somewhat simple emoji searcher in Python. But Python (especially once you start having to install libraries like numpy) isn't that portable. What if we could do this in the browser? That's what this post is going to be all about!

From the methods of the last post, I saved the files `good_vocab.txt` (a list of all the words in the pared-down word2vec model), and `good_weights.npy` (the word vectors of the model). I'll also save the `mapping` variable to `mapping.json` (which will have the word -> emoji mappings).

```javascript
(async () => {
    const mapping = await (await fetch('mapping.json')).json()
    const vocab = (await (await fetch('good_vocab.txt')).text()).split(' ')
    const weights = await (await fetch('good_weights.npy')).arrayBuffer()
})()
```

I'm using the fetch library here because, well, it makes the code very simple. The mapping gets parsed into a dictionary (technically object) of emoji descriptor words -> emoji characters, the words from the model as a string that gets parsed, and the weights as an [`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer) (since it wouldn't parse as UTF-8 or any other encoding).

And in the network panel, we see this little beauty:

![Firefox network panel]({{ "/assets/2018-07-05-network.png" | absolute_url }})

16 seconds to fetch a file! From localhost! I wouldn't even dare think of how slow this would be over the internets.

## Chunking the input

To alleviate this, I'll divide the weights file into chunks. At any given time, all we need are the word vectors for the emoji words and the word being searched up. So I'll make one file for the emoji word vectors, and then divide all the vectors by the first letter of the word they correspond to.

At the same time I'll also divide the vocabulary in this manner

That will give `data/emoji.npy` (emoji word vectors) and `data/emoji.txt` (emoji words); `data/dat-a.npy` (word vectors for words starting with `a` or `A`) and `data/dat-a.txt` (words starting with `a` or `A`), etc. etc.

This gives smaller files, but `dat-s.npy` measures 55 MB! That's still a lot. My data plan would not be pleased. I'll use the same process to make three-character files (so `aaa`, `aab`, `aha`, etc.)

Here's the Python script in case you were interested (variables carried over from previous post):

```python
from itertools import permutations

# Write the two emoji data files
with open('data/emoji.txt', 'w') as f:
    f.write(' '.join(emoji_words))
np.save('data/emoji.npy', emoji_norms)

# Make an array of the first, second, and third letters
# If a word has only one letter, make the second letter a space (so it goes in `a .txt`)
firsts = np.array([v[0].lower() for v in good_vocab])
seconds = np.array([v[1].lower() if len(v) > 1 else ' ' for v in good_vocab])
thirds = np.array([v[2].lower() if len(v) > 2 else ' ' for v in good_vocab])

# Find what characters we'd need to have in the file names
possible = sorted(set(firsts) | set(seconds) | set(thirds))
allowed = possible[:possible.index('~')+1] # Select all up to "~"

fs_forbid = '<>:"/\\|?*' # Characters forbidden by file system

# Save the npy and txt files
def save(letters, indices):
    filename = ''.join( # Escape characters forbidden by file system
        chr(ord(c)+128) if c in fs_forbid else c for c in letters
    )
    with open('data/dat-{}.txt'.format(filename), 'w', encoding='utf-8') as f:
        f.write(' '.join(good_vocab[i] for i in indices[0]))
    np.save('data/dat-{}.npy'.format(filename), good_weights[indices])

threefiles = [] # All two-letter combinations that will have to be split up into three-letter
for fl, sl in permutations(allowed, 2): # Loop 2-letter combinations
    matched = np.logical_and(firsts == fl, seconds == sl) # See which words match the two
    if np.count_nonzero(matched) == 0: # Skip if none match
        continue
    elif np.count_nonzero(matched) < 1000: # Save if fewer than 100 words match
        save(fl + sl, np.where(matched))
        continue
    threefiles.append(fl + sl)
    for tl in allowed: # Split over the third word and save those files
        indices = np.where(np.logical_and(matched, thirds == tl))
        if indices[0].size == 0:
            continue
        save(fl + sl + tl, indices)

with open('data/threefiles.json', 'w') as f:
    json.dump(threefiles, f)
```

This ignores some of the other characters words start with (like non-english characters and emoji), I'll assume no one is going to be searching for words with them anyways as the dataset is mostly English.

Here `allowed` contains the characters ``! " # % & ' ( ) * + , - . / 0 1 2 3 4 5 6 7 8 9 : ; = > @ ^ _ ` a b c d e f g h i i̇ j k l m n o p q r s t u v w x y z ~`` and space.

After running the script, the `data` directory contains several important files:
* `emoji.txt` and `emoji.npy`: emoji vocab and vectors
* `dat-8j.txt`, `dat-8j.npy`, `...`: words and vectors for 2-letter sequences
* `dat-exg.txt` and `dat-exg.npy`, `...`: words and vectors for 3 letter sequences
* `threefiles.json`: JSON-encoded array of two-letter sequences that have been expanded out to three-letter sequences since a lot of words start with these two letters

## Parsing files

Now that the files are chunked, I'll need a function to accept the search term and fetch the matching word and vector files:

```javascript
const mapping = await (await fetch('mapping.json')).json()
const emojiVocab = (await (await fetch('data/emoji.txt')).text()).split(' ')
const emojiWeights = await (await fetch('data/emoji.npy')).arrayBuffer()
const threeFiles = await (await fetch('data/threefiles.json')).json()
const fsForbid = '<>:"/\\|?*'

async function search(term) {
    term = term.replace(/ /g, '_') // Fit the format of the vocab array
    if (term.length == 0) return [] // Passing nothing as input isn't valid

    // Formulate the path to the file Python outputted earlier
    const modTerm = term.replace(/./g, (c) => {
        if (fsForbid.includes(c)) return String.fromCharCode(c.charCodeAt(0) + 128)
        return c
    }).toLowerCase()

    // Find first, second, and third letter, substituting spaces as necessary
    const fl = modTerm[0]
    const sl = modTerm[1] || ' '
    const tl = modTerm[2] || ' '

    const filename = threeFiles.includes(tl) ? fl+sl+tl : fl+sl

    // Fetch the relevant vocab and weights files
    const vocab = (await (await fetch(`data/${filename}.txt`)).text()).split(' ')
    const weights = await (await fetch(`data/${filename}.npy`)).arrayBuffer()
}
```

Much of this code mirrors the python code above used to generate the filenames, as we need to do that too here to locate the files.

Then to compare the search term with the vocab:

```javascript
// Inside function search(term)
let index = vocab.indexOf(term) // Find index by case
if (index == -1) {
    const lowercaseTerm = term.toLowerCase() // Find index without considering case
    index = vocab.findIndex(v => v.toLowerCase() == lowercaseTerm)
}
console.log(index)
```

In the case that the term is not found in a case-senstive search, [`findIndex`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/findIndex) is used to build our own `indexOf`.

And to call the function:

```javascript
search('hello')
```

Now to parse that npy file...

[The docs](https://docs.scipy.org/doc/numpy/neps/npy-format.html) say the first 6 bytes should resolve to the magic string `\x93NUMPY`. Let's verify that:

```javascript
const magicData = new Uint8Array(weights, 0, 6)
let magic = ''
magicData.forEach(c => magic += String.fromCharCode(c))
console.log(magic.charCodeAt(0), magic.substring(1))
```

This gives `147 NUMPY`. `147` is `0x93` in Hex, so the file is valid!

Next comes the major and minor version numbers.

```
const [major, minor] = new Uint8Array(weights, 6, 2)
console.log(`version ${major}.${minor}`)
```

For me that's version `1.0`.

Next comes the header length. It's a 2 byte little-endian unsigned short. Then like the magic number, we can parse the header data into a string:

```javascript
const headerData = new Uint8Array(weights, 10, headerLength)
let header = ''
headerData.forEach(c => header += String.fromCharCode(c))
console.log(header)
```

This gives `{'descr': '<f4', 'fortran_order': False, 'shape': (496, 300), }`. Nice!

A few modifications and we can get this to JSON javascript understands:

```javascript
const parsedHeader = JSON.parse(header
    .replace(/'/g, '"')
    .replace(/False/g, 'false')
    .replace(/True/g, 'true')
    .replace(/\(/g, '[')
    .replace(/\)/g, ']')
    .replace(/, }/g, '}')
)

console.log(parsedHeader)
```

Gives:

```json
{
    "descr": "<f4",
    "fortran_order": false,
    "shape": [496, 300]
}
```

Then comes the data! Let's read the first few floats:

```javascript
const data = new DataView(weights, 10 + headerLength)
console.log(data.getFloat32(0, true))   // 0.024849990382790565
console.log(data.getFloat32(4, true))   // 0.03313332051038742
console.log(data.getFloat32(8, true))   // 0.019124748185276985
console.log(data.getFloat32(12, true))  // 0.011755019426345825
```

Seems reasonable. Now we can organize this into a little function for parsing `.npy` files:

```javascript
function parseNpy(buffer) {
    const magicData = new Uint8Array(buffer, 0, 6)
    let magic = ''
    magicData.forEach(c => magic += String.fromCharCode(c))
    if (magic !== '\x93NUMPY') throw new Error('Invalid magic string')

    const [major, minor] = new Uint8Array(buffer, 6, 2)
    if (major != 1 || minor != 0) throw new Error('Only version 1.0 supported')

    const headerLength = new DataView(buffer).getUint16(8, true)

    return new DataView(buffer, 10 + headerLength)
}
```

To keep things simple I didn't parse the header. I'll hope all the files have the correct ordering and data type.

## Dot products

I'll modify the `emojiWeights` variable to use the `parseNpy` function:

```javascript
const emojiWeights = parseNpy(await (await fetch('data/emoji.npy')).arrayBuffer())
```

Then I'll make a function to take dot products along these two `DataViews`. Here `index1` and `index2` will refer to the index of our word in the vocab array.

```javascript
function dotProduct(view1, index1, view2, index2) {
    let dot = 0
    for (let i = 0; i < 300; i++) { // Each word vector is length 300
        dot += view1.getFloat32(index1 * 300 * 4 + i * 4, true)
             * view2.getFloat32(index2 * 300 * 4 + i * 4, true)
    }
    return dot
}
```

This bit of math works because each 32-bit float takes up 4 bytes. So `i * 4` gives consecutive floats. The `* 300 * 4` spans 300 floats, the number of floats encoding for each word vector.

So within our `search` function that's being called with the value `hello` for now, we can do the following:

```javascript
// index refers to the one defined earlier up in the search functoin
console.log(dotProduct(weights, index, emojiWeights, emojiVocab.indexOf('hi')))
```

Which gives `.65` as the similarity. That's good (somewhat; it should really be higher. But the parsing is working. I promise you)!

To find matching emoji, we need to consider all of them. So we can iterate over the vocab array and find the best matches:

```javascript
// Compute the dot products, also keeping track of the index they correspond
// to in the vocab array
const products = emojiVocab.map((_, emojiIndex) => {
    return [emojiIndex, dotProduct(weights, index, emojiWeights, emojiIndex)]
})
// Sort the products descending by their value (index 1 of the inner arrays)
products.sort((a, b) => a[1] - b[1])
products.reverse()

// Output the top 10 results
for (let i = 0; i < Math.min(products.length, 10); i++) {
    const [index, dotprod] = products[i]
    console.log(dotprod, emojiVocab[index])
}
```

This gives:

```
1.0000000161893414 hello
0.6548984408213944 hi
0.6399055901641588 goodbye
0.5544619264442038 hug
0.4897352942405049 smiling
0.4730159033812229 namaste
0.47205641057925707 chatting
0.45117140669361094 hugs
0.44649946734886803 smile
0.433194592057971 xox
```

Or to find the top ten emoji rather than words:

```javascript
const emoji = [] // Keep track of emoji already outputted to avoid duplicates
for (let i = 0; i < products.length && emoji.length < 10; i++) {
    const [index, dotprod] = products[i]
    for (let emote of mapping[emojiVocab[index]]) {
        // If we still need more emoji and this is a new one, console.log it
        if (emoji.length < 10 && !emoji.includes(emote)) {
            emoji.push(emote)
            console.log(emote, emojiVocab[index], dotprod.toFixed(3))
        }
    }
}
```

Which gives:

```
👋 hello 1.000
🤗 hug 0.554
😈 smiling 0.490
😙 smiling 0.490
🙂 smiling 0.490
🙏 namaste 0.473
🗨 chatting 0.472
💬 chatting 0.472
😄 smile 0.446
😛 smile 0.446
```

Yay! Next post I'll be looking at the efficiency of the code so far.
