---
layout: post
title:  "Emoji vectors and optimization. Oh my! (Part 1: Word2Vec and Python)"
date:   2018-07-05
---

# Emoji vectors and optimization. Oh my! (Part 1: Word2Vec and Python)

A while back ago, the Slack bot [EmojiBot](https://slack.com/apps/A0HKW0SFK-emojibot) went offline, endangering my workflow. You could ask it to find an emoji by sending a query like `@emojibot electricity`, and it would reply with emoji such as :electric_plug:, :zap:, :bulb:, and :battery:. But all was okay because in the meantime, I built up my own replacement that was a [Python http server](https://github.com/rianadon/Emoji-Suggester/tree/master/emojiserver) connected to a lovely little [Hubot](https://hubot.github.com/) instance. This post details how I went about doing that.

Before I move on, a few basics on the overarching question: how does one match up text to emoji?

One great project I encountered was [emojilib](https://github.com/muan/emojilib). The project contains a mapping of emoji to a few descriptive key words each. You can find a search interface for it [here](http://emoji.muan.co). It works spectacularly for most emoji, but once you pull out a thesaurus things get worse. For example: `ghost` finds :ghost:, but `ghoul` turns up nothing. `Happy` finds a long list of emoji (:grinning:, :grin:, :joy:, and :smiley: to name a few), but `ecstatic` finds none. And nothing comes up for `Ryan`, my very own name, either!

We can do better. Google has word vectors of 300 million words in its Google News word2vec dataset! Surely there must be a way to utilize this. And there is. There are already methods to make a word2vec model out of emoji, such as the one detailed by [this paper](https://arxiv.org/abs/1609.08359). And while the word2vec training methods are amazingly ingenious and interesting, I decided to not dive into the depths of AI deep neural network machine learning and rather play around with Google's word embeddings and emojilib.

With word2vec, you can find the similarity of two words by finding the angle between their two vectors. This is the same as taking the arc cosine dot product of their unit vectors, However, we can deal without the trigonometry and just take dot products, which gives similarities between 0 and 1.

With this method, I can build a way to relate fancy words (e.g. ecstatic) -> simple words (e.g. happy) -> emoji, and will be able to build a pretty good emoji searcher!

## Building a Map

Google's word vectors are for words, not emoji. So I need a way to translate words to emoji. Luckily, this is easy to do with emojilib and a little Python:

```python
with open('emojilib.json', 'r', encoding='utf-8') as f:
    emojis = json.load(f) # Assume emojilib is downloaded as emojilib.json

mapping = defaultdict(list) # A dict where each item defaults to []

for emoji_name, emoji in emojis.items():
    mapping[emoji_name].append(emoji['char'])
    for part in emoji_name.split('_'):
        mapping[part].append(emoji['char'])
    for keyword in emoji['keywords']:
        mapping[keyword].append(emoji['char']) # Add extra terms into the mapping

print(mapping) # The generated mapping of words -> emoji
```

Then we get a nice mapping like this for categories:

```
flag: 🇬🇧 🇸🇰 🇹🇨 🇲🇰 🇳🇫 🇧🇯 🇹🇷 🇬🇷 🇯🇴 🇰🇲 🇲🇾 🇹🇿 🇵🇭 🇨🇩 🇩🇲 🇬🇳 🇱🇷 🇨🇻 🇻🇬 🇲🇿 🇵🇱 🇻🇦
banner: 🇬🇧 🇸🇰 🇹🇨 🇲🇰 🇳🇫 🇧🇯 🇹🇷 🇬🇷 🇯🇴 🇰🇲 🇲🇾 🇹🇿 🇵🇭 🇨🇩 🇩🇲 🇬🇳 🇱🇷 🇨🇻 🇻🇬 🇲🇿 🇵🇱
country: 🇬🇧 🇸🇰 🇹🇨 🇲🇰 🇳🇫 🇧🇯 🇹🇷 🇬🇷 🇯🇴 🇰🇲 🇲🇾 🇹🇿 🇵🇭 🇨🇩 🇩🇲 🇬🇳 🇱🇷 🇨🇻 🇻🇬 🇲🇿 🇵🇱
nation: 🇬🇧 🇸🇰 🇹🇨 🇲🇰 🇳🇫 🇧🇯 🇹🇷 🇬🇷 🇯🇴 🇰🇲 🇲🇾 🇹🇿 🇵🇭 🇨🇩 🇩🇲 🇬🇳 🇱🇷 🇨🇻 🇻🇬 🇲🇿 🇵🇱
woman: 💑 👩‍👦 🧗‍♀️ 🧗‍♀️ 🚣‍♀️ 🚣‍♀️ 👩‍⚕️ 👩‍⚕️ 👩‍👩‍👧 👩‍👩‍👧 🧜‍♀️ 💂‍♀️ 🤦‍♀️ 🤦‍♀️ 👩‍🎨 👩‍🎨
nature: 🦑 🐩 🎐 🌕 🌷 🙈 🌾 🐺 🐑 🏔 🐊 🐏 🐄 🐡 🌚 🌓 🐘
man: 💑 🧝‍♂️ 🧝‍♂️ 🤷‍♂️ 🤷‍♂️ 👱 👱 🤽‍♂️ 💁‍♂️ 💁‍♂️ 🚣 🧘‍♂️ 🧘‍♂️ 🙋‍♂️ 🙋‍♂️
face: 😄 😤 😛 🤤 🤤 😢 👂 😕 😀 😬 😷 😁 🌚 🤬 😣 🙃 🙃 🤣
animal: 🦑 🐩 🙈 🐺 🐑 😽 🐊 🐈 🐏 🐄 🐡 🕊 🐘 🦈 🦅 🐤 🦒
human: 💑 👩‍👦 👩‍⚕️ 👩‍👩‍👧 👫 💁‍♂️ 👩‍🎨 👩‍👩‍👧‍👧 👩‍👧‍👧 👨‍🎤 👩‍🌾 ⛹ 👨‍👦 👵 🏊 👥 👨‍✈️ 👨‍❤️‍👨 👩‍👧‍👦
```

And this for individual emoji:

```
lost: 🏳
moving: 📦
sterling: 💷
yummy: 😋
vegas: 🎰
suspension: 🚟
eat: 🍽
bolivarian: 🇻🇪
saucer: 🛸
second: 🥈
```

Wonderful! Now onto utilizing this!

## Short notice: Making `gensim` more efficient

I use the Python `gensim` library for parsing Google's `.bin` file. However, when loading the dataset, `gensim` stores it all in memory. WIth my little 8 GB laptop already running a few browsers, things get a little tight (looking at you Electron). Actually extremely tight. Like Windows freezes as it tries to move memory pages to the disk in panic. Not to great. To rememdy this you can move all the vectors to a [`memmap`](https://docs.scipy.org/doc/numpy-1.14.0/reference/generated/numpy.memmap.html):

`gensim` stores its data in two places: an array of all the words (`in`, `at`, etc.) and a 2D array of vectors, each being mapped to its word by the index. The code below saves both of these for later use (to avoid loading the full model into memory each time:

```python
from gensim.models.keyedvectors import KeyedVectors
model = KeyedVectors.load_word2vec_format('GoogleNews-vectors-negative300.bin', binary=True)

with open('vocab.json', 'w') as f:
    json.dump(model.index2word, f) # Save the word strings

model.init_sims(replace=True) # Calculate all the unit vectors
norms = np.memmap('normsmemmap.dat', dtype=model.wv.vectors_norm.dtype,
                  mode='w+', shape=model.wv.vectors_norm.shape)
norms[:] = model.wv.vectors_norm # Write the normed vectors to the memmap
del model # Discard the model from memory
```

And to load it:

```python
with open('vocab.json', 'r') as f:
    vocab = json.load(f)
norms = np.memmap('normsmemmap.dat', dtype=np.float32).reshape((-1, 300))
```

## Filtering the map

While Google's dataset has 3 million words, emoji names can still be obscure enough to avoid appearance in the English language. For example, `shipit` is not yet a recognized as a word. Neither is `dog2`. Country names as lowercase also aren't included. That requires a little extra parsing work:

```python
lowercasedvocab = [word.lower() for word in vocab]
vocab_set = set(vocab) # For faster searching
lower_vocab_set = set(lowercasedvocab) # Again, faster searching
for word in list(mapping.keys()):
     if word not in vocab_set:
        # Check to see if tje word appears as a different case
        if word.lower() in lower_vocab_set:
            cased_word = vocab[lowercasedvocab.index(word)]
            mapping[cased_word] = mapping[word]
        # Remove the word from the mapping
        del mapping[word]
```

This process performs operations such as the following (✍ for rename, ✖ for remove):

```
✍ papua → Papua
✖ couple_with_heart_woman_man
✖ funeral_urn
✖ man_elf
✍ kazakhstan → Kazakhstan
✖ drooling_face
✖ ideograph
✖ climbing_woman
✖ medal_military
✖ blonde_man
✖ straight face
✍ norfolk_island → NORFOLK_ISLAND
✖ rowing_woman
✖ oncoming_police_car
✖ highfive
✍ full_moon → Full_Moon
```

Here utilizing `set` makes a huge difference. At least a 5x gain in speed. It's a lot faster to search a hash table than to iterate through a 3 million word array.

## Filtering the word vectors

Google's dataset is HUGE. It takes a ton of memory and a ton of time to load it, and in reality we only care about emoji. If words aren't similar to emoji names, we don't need them.

The code below uses `np.inner`, which is a nice way of computing dot products between the 3mil x 300 word2vec vocab and 2854 x 300 emoji vocab and giving a 3mil x 2854 array of the dot product results. You may see from this that we could transpose the emoji vocab array so we can do a matrix multiplication (`@` in numpy) between the new 3mil x 300 and 300 x 2854 arrays, but `np.inner` is a little simpler to call in my opinion.

Also, computing this 3mil x 2854 array in go would take a *lot* of memory. So we do it in chunks and write these chunks to a `memmap`. Using `amax` to find the maximum dot product for each word (i.e. find the closest each word is to our emoji vocab) also decreases the memory needed.

```python
vocab_norms = np.array([norms[vocab.index(word)] for word in mapping])

dp = np.memmap('dpmap.dat', dtype=norms.dtype, mode='w+', shape=(norms.shape[0],))
CHUNKSIZE = 1000
for s in range(0, norms.shape[0], CHUNKSIZE):
    e = s+CHUNKSIZE
    dp[s:e] = np.amax(np.inner(norms[s:e], vocab_norms), axis=1)
```

Now we can find things like the words least similar to our emoji vocab:

```
          Word              Similarity
-------------------------   -----------
HuMax_IL8_TM                0.045590762
J.Gordon_##-###             0.05180111
By_DOUG_HAIDET              0.056708228
G.Biffle_###-###            0.068331175
K.Kahne_###-###             0.08244177
HuMax_TAC_TM                0.08358895
mso_para_margin_0in         0.08385273
Nasdaq_NASDAQ_TRIN          0.08743682
BY_ROBERTO_ACOSTA           0.08953415
Globalization_KEY_FACTORS   0.09093615
```

At least it seems Google `#`ed out personal information. That's nice. I have no idea why people's names ended up in the dataset in the first place though.

And things like the most similar words:

```
     Word         Similarity
---------------   ----------
Senegal           1.0000011
bearded           1.000001
drink             1.000001
urn               1.000001
industry          1.0000008
fly               1.0000008
training          1.0000008
organizing        1.0000008
Macao             1.0000007
dark_sunglasses   1.0000007
```

These dot products should never be greater than one. I'll blame it on floating point imprecision.

I'll choose a random cutoff (let's say `0.5`), and keep only the words with a similarity greater than or equal to `0.5`:

```python
good_indices = np.where(dp > 0.5)
good_vocab = np.array(vocab)[good_indices]
with open('good_vocab.txt', 'w', encoding='utf-8') as f:
    for word in good_vocab:
        f.write(word + ' ')
good_norms = norms[good_indices]
np.save('good_weights.npy', good_norms)
```

This yields about a 570 MB file (you can cut that down to 144 MB with a threshold of `0.6`). That's somewhat more manageable! Gzipped it's about 440 MB, which is still on the large side.

## Using the filtered vectors

Now to use the data:

```python
# Reform the list of words and normalized vectors pointing to words in the emoji map
lookup = lambda word: good_weights[good_vocab.index(word)]
emoji_words = list(mapping.keys())
emoji_norms = np.array([lookup(word) for word in emoji_words])

# Calculate similarities (dot products) between
dot = np.dot(emoji_norms, lookup('ecstatic'))
matches = np.argpartition(dot, -10)[-10:]
sortedmatches = matches[np.argsort(dot[matches])][::-1]
for index in sortedmatches:
    print(dot[index], emoji_words[index], ' '.join(mapping[emoji_words[index]]))
```

This gives:

```
0.6626913 happy 😄 😀 😁 😃 😊 😹 😆 😋 🌈 😂 😉 😺 😅
0.57641125 disappointed 😥 🙁 😞 😞
0.5699152 surprised 😲
0.5558953 glad 😆
0.55013615 shocked 🤯
0.543222 astonished 😲 😲
0.54264677 proud 😤
0.53669024 stunned 😧
0.5175118 awesome ✨ ❇️ 🌟 👍
0.51358485 relieved 😥 😌 😌
```

Not bad! I've never heard of `disappointed` being a synonyms for `ecstatic`, but everything else makes sense.

In part 2, I'll move to JavaScript so we can get a nicer emoji search in the browser.
