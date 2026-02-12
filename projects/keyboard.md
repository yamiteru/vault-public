# Keyboard

The goal of the project is to let a genetic algorithm design a computer keyboard letter-only (no symbols or special keys) layout.

The layout will be based on uni/bi/trigram frequencies computed from Fineweb (for english) and The Stack V2 (for programming languages).

User defines their keyboard size in terms of row and column count and provides effort scores for each finger type (index, middle, ring, pinkie).

In the first step a simple algorithm chooses the best key positions for the given number of keys (26 for English).

We define a chromosome as an encoded sequence of ASCII character codes in a byte array based on the provided keyboard size and the chosen least-effort keys (in a top-to-bottom and left-to-right fashion).

For example we can have a keyboard layout of 12 columns and 4 rows with selected fingers based on effort scores from the simple algorithm:

```
[
    [_, _, _, m, i, _, _, i, m, _, _, _],
    [_, p, r, m, i, i, i, i, m, r, p, _],
    [p, p, r, m, i, _, _, i, m, r, p, p],
    [_, p, _, _, _, _, _, _, _, _, p, _],
]
```

This is how it would look if we mapped a layout onto the grid:

```
[
    [_, _, _, a, b, _, _, c, d, _, _, _],
    [_, e, f, g, h, i, j, k, l, m, n, _],
    [o, p, q, r, s, _, _, t, u, v, w, x],
    [_, y, _, _, _, _, _, _, _, _, z, _],
]
```

Which would then be encoded into a chromosome like this:

```
[97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113, 114, 115, 116, 117, 118, 119, 120, 121, 122]
```

The genetic algorithm will start with these constants:
- Parent count `Pc` (5) which defines how many parents are needed for breeding
- Group count `Gc` (10) which defined how many groups are selected for breeding
- Population multiplier `Pm` (2) which defined how many times should `Gc` be multiplied to get `Po`
- Population size `Po` (100) `Gc * Pm` which defined the numbed of items in each generation
- Tournament size `Ts` (2) which defined how many items compete with each other
- Tournament count `Tc` (50) `Gc * Pc` which defined how many tournaments are played in each generation

In the first generation we seed `Po` (100) items with random chromosomes.

In each generation for every item we compute a fitness score defined as:
- ✅ Hand unigram balance
- ✅ Hand bigram balance
- ✅ Hand trigram balance
- ✅ Finger unigram balance
- ✅ Finger bigram balance
- ✅ Finger trigram balance
- ✅ Bigram inward rolls
- ✅ Bigram rolls with strong fingers
- ✅ Bigram rolls with strong fingers
- ✅ Alternating bigrams without row change
- ✅ Alternating trigrams without row change
- ✅ Alternating trigrams where one hand does inward roll bigram with strong fingers and the other a single key press with a strong finger
- .. possibly more

After that we let them play `Tc` (50) tournaments where one item out of `Ts` (2) items is selected based on fitness score.

These items are then randomly grouped into `Gc` (10) groups where each group consists of `Pc` (5) items.

For each group we calculate a weighted intersection which is an intersection where 2+ parents agree on certain letters in certain positions with the added inofrmation of how many of them agree.

When creating a child we copy-paste the intersection values (without the counts) in the child.

After that if there are gaps we sequentially ho through all parents and for each gap at position `i` find a letter in any of the parents also at the position `i` and insert it in the child (if all letters in parents at the position `i` would end up being duplicate then we simply insert a random letter from a free list which is a list of letters that have not been added into the child yet).

This crossover produces a child with zero percent mutations.

We then copy-paste this child `Pc - 1` times and add `100 / (Pc - 1) * i` percent mutations.

Mutations are done by flipping 2 bytes in the chromosome.

The 2 bytes are selected based on the count score from the intersection (with positions that had no intersection having a null value and count 0).

We select the positions with the lowest count first and keep selecting higher counts as we start doing more mutations.

---

I should add info about Simulated Annealing which would:
1) change the upper bound of mutations from 100% to let's say 5%
2) change the mutation key selection from being global to being local

I should also define fitness metrics better and add weights.

An important thing to note is that trigram weights should be defined as proportional to the sum of percent of the top 50 bigrams and trigrams (there's less bigrams so the top 50% bigrams is responsible for more percent of the total than trigrams).
