---
layout: post
title: "NEAT and NES Algorithms"
description: The Korean government provides a cryptography library for its GPKI program. This library contains two symmetric encryption algorithms, NEAT and NES, which have not been published. We have reverse engineered these algorithms and provide a description and working implementations.
modified: 2017-06-19
category:
  - Research
featured: true
mathjax: true
---

The South Korean Ministry of the Interior provides a cryptography library for its GPKI program. This library contains two symmetric encryption algorithms, NEAT and NES, which have not been published. We reverse engineered these algorithms, and are providing descriptions and working implementations.

(Korean version of this post can be found [here](/research/korean/reversing-crypto-from-libraries).)

## NEAT

National Encryption Algorithm (NEAT) is a block cipher with a 128-bit block size and 128-bit key size. It is believed to have been developed in 1997 by KISA. It takes inspiration from the [IDEA](https://en.wikipedia.org/wiki/International_Data_Encryption_Algorithm) and [RC5](https://en.wikipedia.org/wiki/RC5) block ciphers. It operates on the two 64-bit halves of the block, and combines portions of the two halves with a bitwise XOR.

### Operation

NEAT has 12 full rounds and one half round (no mix). Each round splits the block into 16-bit integers and performs several operations on these integers. Similar to [IDEA](https://en.wikipedia.org/wiki/International_Data_Encryption_Algorithm), NEAT interleaves operations from different groups:

* Bitwise XOR (⊕)
* Addition modulo 2<sup>16</sup> (⊞)
* Multiplication modulo 2<sup>16</sup>+1, with 0x0000 treated as 2<sup>16</sup> (⊙)

NEAT borrows an addtional operation, data dependent rotation, from [RC5](https://en.wikipedia.org/wiki/RC5):

* Rotation (&#8920;)

<img src="{{ site.url }}/images/2017-06-19/neat_f.png" style="width: 49%">
<img src="{{ site.url }}/images/2017-06-19/neat_fi.png" style="width: 49%">
<figcaption><i>F</i> and <i>F<sup>-1</sup></i> round functions</figcaption>

The 128-bit block is split into two 64-bit halves. The left half is fed into *F* and the right half is fed into *F<sup>-1</sup>*. The round functions split each 64-bit half into four 16-bit integers, perform operations, and then concatenate back into a 64-bit block. Each round splits the 64-bit round key into four 16-bit subkeys, *K<sub>n</sub>*, which are used by the multiplication operation.

<figure>
<img src="{{ site.url }}/images/2017-06-19/neat_round.png">
<figcaption>NEAT Round</figcaption>
</figure>

The MIX function does a bitwise XOR of some bits from the left half and the right half. The set of bits mixed together depend on the round key. 

The last round is a half round without the MIX function. Instead, the left halves and right halves are swapped. This is important because it allows us to use the same code for encryption and decryption.

<figure>
<img src="{{ site.url }}/images/2017-06-19/neat_half_round.png">
<figcaption>NEAT Half-Round</figcaption>
</figure>

### Key Schedule

The NEAT algorithm uses 13 round keys, one per full round and one for the last round. These round keys are generated using the same algorithm as encryption. The provided master key is treated as a 128-bit data block, and the round key is fixed:

{% highlight c %}
const uint16_t Ks[] = { 0xFD56, 0xC7D1, 0xE36C, 0xA2DC };
{% endhighlight %}

Each round will generate two 64-bit round keys, so after 6.5 rounds we will have enough round keys. The mix function for each round is determined by *K<sub>1</sub>* ^ *K<sub>4</sub>* mod 128.

#### Decryption

As mentioned previously, we can use the same code for encryption and decryption. The only difference is that after generating the round keys for encryption we need to transform them into decryption round keys.

Since *F* and *F<sup>-1</sup>* use different *K<sub>n</sub>* we need to swap (*K<sub>1</sub>*, *K<sub>2</sub>*) with (*K<sub>3</sub>*, *K<sub>4</sub>*). Also, we need to reverse the order of the round keys so that we use the round keys from the last round (encryption) during the first round (decryption). Lastly, in order to reverse the multiplication operation we need to calculate the multiplicative inverse of each subkey.

## NES

National Encryption Standard (NES) is a block cipher with a 256-bit block size and 256-bit key size, developed in 2003. Similar to AES, it is a substitution-permutation network (SPN) and operates on each 8-bit byte of the block. It appears to be inspired by Anubis and KHAZAD, which are involutional SPN ciphers. It also has some similarities to the publically released ARIA block cipher.

### Operation

NES operates on each byte as an element in [Rijndael's finite field](https://en.wikipedia.org/wiki/Finite_field_arithmetic#Rijndael.27s_finite_field). Bitwise XOR is used as the addition operator, and multiplication is modulo the irreducibble polynomial $$ x^8+x^4+x^3+x+1 $$. It uses a S-box for substitution and the permutation can be viewed as a 8x8 matrix multiplication.

<img src="{{ site.url }}/images/2017-06-19/nes_full.png" style="width: 32%">
<img src="{{ site.url }}/images/2017-06-19/nes_round.png" style="width: 32%">
<img src="{{ site.url }}/images/2017-06-19/nes_final_round.png" style="width: 32%">

<figcaption>NES Encryption  |  NES Round  |  NES Final Round</figcaption>

NES has 11 full rounds and a last round without permutation. Each round adds a 128-bit round key to the state. Before the first round, an initial round key is added to the initial state. Depending on whether the full round is even or odd, it will use different matrix for permutation.

1. Initial Round (Round 0)
   1. AddRoundKey
2. Rounds (Round 1~11)
   1. Substitution
   2. Transposition
   3. MixColumns (permutation)
   4. AddRoundKey
3. Final Round
   1. Substitution
   2. Transposition
   3. AddRoundKey

Assuming an input state of the form $$ [a_0, a_1, a_2, a_3, a_4, ... a_{n-1}] $$, the substitution and transposition operations can be viewed as matrix operations resulting in a 8x4 matrix. The function S applies substitution on a byte using the S-box.

$$
\begin{bmatrix}
  S(a_{0}) &  S(a_{8}) & S(a_{16}) & S(a_{24}) & S(a_{4}) & S(a_{12}) & S(a_{20}) & S(a_{28}) \\
  S(a_{1}) &  S(a_{9}) & S(a_{17}) & S(a_{25}) & S(a_{5}) & S(a_{13}) & S(a_{21}) & S(a_{29}) \\
  S(a_{2}) &  S(a_{10}) & S(a_{18}) & S(a_{26}) & S(a_{6}) & S(a_{14}) & S(a_{22}) & S(a_{30}) \\
  S(a_{3}) &  S(a_{11}) & S(a_{19}) & S(a_{27}) & S(a_{7}) & S(a_{15}) & S(a_{23}) & S(a_{31})
\end{bmatrix}
^{\LARGE{T}}
=
\begin{bmatrix}
  b_{0} &  b_{8} & b_{16} & b_{24} \\
  b_{1} &  b_{9} & b_{17} & b_{25} \\
  b_{2} &  b_{10} & b_{18} & b_{26} \\
  b_{3} &  b_{11} & b_{19} & b_{27} \\
  b_{4} &  b_{12} & b_{20} & b_{28} \\
  b_{5} &  b_{13} & b_{21} & b_{29} \\
  b_{6} &  b_{14} & b_{22} & b_{30} \\
  b_{7} &  b_{15} & b_{23} & b_{31}
\end{bmatrix}
$$

In a full round, the resultant matrix can be transformed by matrix multiplication with one of the two following matrices. In a half round, no permutation is performed, which is the same as multiplication with the identity matrix.

$$
\begin{bmatrix}
    3 & 1 & 1 & 2 & 0 & 0 & 0 & 0 \\
    2 & 3 & 1 & 1 & 0 & 0 & 0 & 0 \\
    1 & 2 & 3 & 1 & 0 & 0 & 0 & 0 \\
    1 & 1 & 2 & 3 & 0 & 0 & 0 & 0 \\
    0 & 0 & 0 & 0 & 3 & 1 & 1 & 2 \\
    0 & 0 & 0 & 0 & 2 & 3 & 1 & 1 \\
    0 & 0 & 0 & 0 & 1 & 2 & 3 & 1 \\
    0 & 0 & 0 & 0 & 1 & 1 & 2 & 3 \\
\end{bmatrix}
\begin{bmatrix}
  b_{0} &  b_{8} & b_{16} & b_{24} \\
  b_{1} &  b_{9} & b_{17} & b_{25} \\
  b_{2} &  b_{10} & b_{18} & b_{26} \\
  b_{3} &  b_{11} & b_{19} & b_{27} \\
  b_{4} &  b_{12} & b_{20} & b_{28} \\
  b_{5} &  b_{13} & b_{21} & b_{29} \\
  b_{6} &  b_{14} & b_{22} & b_{30} \\
  b_{7} &  b_{15} & b_{23} & b_{31}
\end{bmatrix}
=
\begin{bmatrix}
  c_{0} &  c_{8} & c_{16} & c_{24} \\
  c_{1} &  c_{9} & c_{17} & c_{25} \\
  c_{2} &  c_{10} & c_{18} & c_{26} \\
  c_{3} &  c_{11} & c_{19} & c_{27} \\
  c_{4} &  c_{12} & c_{20} & c_{28} \\
  c_{5} &  c_{13} & c_{21} & c_{29} \\
  c_{6} &  c_{14} & c_{22} & c_{30} \\
  c_{7} &  c_{15} & c_{23} & c_{31}
\end{bmatrix}
$$

<center>or</center>

$$
\begin{bmatrix}
    3 & 7 & 5 & 2 & 5 & 4 & 2 & 3 \\
    3 & 3 & 7 & 5 & 2 & 5 & 4 & 2 \\
    2 & 3 & 3 & 7 & 5 & 2 & 5 & 4 \\
    4 & 2 & 3 & 3 & 7 & 5 & 2 & 5 \\
    5 & 4 & 2 & 3 & 3 & 7 & 5 & 2 \\
    2 & 5 & 4 & 2 & 3 & 3 & 7 & 5 \\
    5 & 2 & 5 & 4 & 2 & 3 & 3 & 7 \\
    7 & 5 & 2 & 5 & 4 & 2 & 3 & 3 \\
\end{bmatrix}
\begin{bmatrix}
  b_{0} &  b_{8} & b_{16} & b_{24} \\
  b_{1} &  b_{9} & b_{17} & b_{25} \\
  b_{2} &  b_{10} & b_{18} & b_{26} \\
  b_{3} &  b_{11} & b_{19} & b_{27} \\
  b_{4} &  b_{12} & b_{20} & b_{28} \\
  b_{5} &  b_{13} & b_{21} & b_{29} \\
  b_{6} &  b_{14} & b_{22} & b_{30} \\
  b_{7} &  b_{15} & b_{23} & b_{31}
\end{bmatrix}
=
\begin{bmatrix}
  c_{0} &  c_{8} & c_{16} & c_{24} \\
  c_{1} &  c_{9} & c_{17} & c_{25} \\
  c_{2} &  c_{10} & c_{18} & c_{26} \\
  c_{3} &  c_{11} & c_{19} & c_{27} \\
  c_{4} &  c_{12} & c_{20} & c_{28} \\
  c_{5} &  c_{13} & c_{21} & c_{29} \\
  c_{6} &  c_{14} & c_{22} & c_{30} \\
  c_{7} &  c_{15} & c_{23} & c_{31}
\end{bmatrix}
$$

In either case, the resultant matrix is stored as $$ [c_0, c_1, c_2, c_3, c_4, ... c_{n-1}] $$ and is bitwise XOR with the round key.

### Key Schedule

The key schedule is based on the round function used in decryption, using the inverse S-box and one of the inverse P-box (i.e. no concept of even/odd rounds). The state is initialized with the supplied 256-bit master key. The state is viewed as eight 32-bit integers, *b<sub>0</sub>* ... *b<sub>7</sub>*.

Each key schedule round operates (substitution, permutation, and add round key) on 32 bits of the state, *b<sub>j</sub>*, with *b<sub>j+1</sub>* as the round key. The 32-bit output of each key schedule round, *b'<sub>j</sub>*, is bitwise XOR with other bits in the state to generate the full 128-bit encryption round key: [ *b'<sub>j</sub>*, *b'<sub>j</sub>* ^ *b<sub>j+2</sub>*, *b'<sub>j</sub>* ^ *b<sub>j+3</sub>*, *b'<sub>j</sub>* ^ *b<sub>j+4</sub>*].

#### Decryption
NES is an almost involutional SPN block cipher. Encryption and decryption share the same code and differ only in the key schedule, S-box, and P-box. During decryption, the following inverse matrices are used for the even and odd rounds:

$$
\begin{bmatrix}
    9 & 14 & 11 & 13 & 0 & 0 & 0 & 0 \\
    13 & 9 & 14 & 11 & 0 & 0 & 0 & 0 \\
    11 & 13 & 9 & 14 & 0 & 0 & 0 & 0 \\
    14 & 11 & 13 & 9 & 0 & 0 & 0 & 0 \\
    0 & 0 & 0 & 0 & 9 & 14 & 11 & 13 \\
    0 & 0 & 0 & 0 & 13 & 9 & 14 & 11 \\
    0 & 0 & 0 & 0 & 11 & 13 & 9 & 14 \\
    0 & 0 & 0 & 0 & 14 & 11 & 13 & 9 \\
\end{bmatrix}
$$

$$
\begin{bmatrix}
    0x45 & 0x72 & 0x13 & 0xDF & 0x54 & 0x53 & 0x50 & 0x5A \\
    0x5A & 0x45 & 0x72 & 0x13 & 0xDF & 0x54 & 0x53 & 0x50 \\
    0x50 & 0x5A & 0x45 & 0x72 & 0x13 & 0xDF & 0x54 & 0x53 \\
    0x53 & 0x50 & 0x5A & 0x45 & 0x72 & 0x13 & 0xDF & 0x54 \\
    0x54 & 0x53 & 0x50 & 0x5A & 0x45 & 0x72 & 0x13 & 0xDF \\
    0xDF & 0x54 & 0x53 & 0x50 & 0x5A & 0x45 & 0x72 & 0x13 \\
    0x13 & 0xDF & 0x54 & 0x53 & 0x50 & 0x5A & 0x45 & 0x72 \\
    0x72 & 0x13 & 0xDF & 0x54 & 0x53 & 0x50 & 0x5A & 0x45 \\
\end{bmatrix}
$$

The decryption key schedule can be generated from the encryption key schedule by first reversing the order of the round keys. Then, for each full round key, apply the appropriate inverse matrix to the round key. This effectively swaps the order of the AddRoundKey and MixColumns steps in the round function.

## Implementations

On the [GitHub repository](https://github.com/theori-io/kr-crypto), you can find working C and Python implementations of both NEAT and NES. Since these algorithms have not been formally published by their authors, it is unknown if they are covered by any patents. The implementations have been tested against the library we used as a reference. 