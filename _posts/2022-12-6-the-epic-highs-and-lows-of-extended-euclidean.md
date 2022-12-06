---
tags:
- Math
- Crypto
---
<b>This is a draft for testing... need to finish writing this and figure out inline LaTeX</b>

As the fall semester ends - and with it, Guha's cryptography class - I've been looking for ways to keep learning. I started working my way through a site called [CryptoHacks](https://cryptohack.org/challenges/). One of the early challenges involves calculating modular inverses.

You can do this with Python builtins:
```python
pow(a, -1, n)
```
But I figured this was a good opportunity to try writing my own mod inverse from scratch.
This post will contain a quick crash course in modular arithmetic + some light number theory - if you know how arithmetic works, you should be fine.

<h2>So what's a mod inverse?</h2>

Let's start with what "mod" means.
In computing, "mod" is an operation that returns the remainder of integer division.
It's usually represented with a percent sign "%".
So \(14 \textrm{mod} 5\) is 4, since if you try to split 14 objects into groups of 5, you'll have 4 left.
Modular arithmetic is widely used in cryptography. Partially because we can restrict large numbers to a resonable side, and partially because of [some properties](https://en.wikipedia.org/wiki/Discrete_logarithm) that make a modulus operation hard to undo.

More generally, in math "mod" is a congruence relation.
Congruence relations mean "these two things are equal under this condition."
For example, we can say
<div>$14 \equiv 4\ (\textrm{mod}\ 5)$</div>

Obviously 14 and 4 aren't equal, but *under the condition* mod 5 they are (since 14 mod 5 is 4 and 4 mod 5 is also 4).
There's a few rules for how we can work with congruence relations.
Unlike equalities, we can't just apply any operations to both sides and have the condition still hold true.

<p>For example, what if we divide both sides of the congruence relation by 5?</p>
<div>$2.8 \equiv 0.8\ (\textrm{mod}\ 5)$</div>
<p>This is meaningless - we can't do integer division (which mod is based on) with decimal numbers.</p>

However, adding to both sides is fine.
<div>$14 + 3 \equiv 4 + 3\ (\textrm{mod}\ 5)$</div>
<div>$17 \equiv 7\ (\textrm{mod}\ 5)$</div>
<div>$17\ \textrm{mod}\ 5 = 2$</div>
<div>$7\ \textrm{mod}\ 5 = 2$</div>
Additionally, subtracting from and multiplying by both sides is fine. This is left as an exercise to the reader.
This makes an interesting problem. What if we want to solve this congruence relation?

<div>$3x \equiv 2\ (\textrm{mod}\ 5)$</div>
<p>We can't divide both sides by 3, so how can we solve for x?
If we try numbers, we can find that an x value of 4 works.</p>
<div>$3*4 = 12 \equiv 2\ (\textrm{mod}\ 5)$</div>
<p></p>

<h2>How can we solve this without guessing?</h2>

This is where the concept of the modular inverse comes in.
Let's say we have an integer a and a modulus n.
The mod inverse a' of a satisfies this congruence relation: a * a' = 1 (mod n)
The value of a' depends both on the value of a and n.
So an integer a has a different inverse for different moduli.
This inverse exists iff (if and only if) a and n are coprime (they do not share any factors). The proof for this is beyond the scope of this post but there's plenty of [resources](https://en.wikipedia.org/wiki/Modular_multiplicative_inverse) for that.

How does this help us solve the equation?
<div>$3x \equiv 2\ (\textrm{mod}\ 5)$</div>
Well, if we know that the mod inverse of 3 mod 5 is 2, then we can multiple both sides of the equation by 2:
<div>$3 * 2 * x \equiv 2 * 2\ (\textrm{mod}\ 5)$</div>
By the definition of mod inverse, a * a' = 1
<div>$x \equiv 2 * 2\ (\textrm{mod}\ 5) = 4$</div>

Hopefully this gives you an idea of why modular inverses are useful - they bascially replace the concept of division in modular arithmetic.


<h2>So how can we find a modular inverse?</h2>
The Extended Euclidean Algorithm of course!
The Euclidean algorithm is a method of finding the GCD (greatest common divisor) of two numbers.
It relies on the fact that if you have two numbers a and b with a > b,
<div>$GCD(a,\ b) = GCD(a - b,\ b)$</div>
In other words, if we replace the larger number with the difference between the numbers, the GCD stays the same!
We can take this a step further and say
<div>$GCD(a,\ b) = GCD(a\ \% \ b,\ b)$</div>
If a - b is still greater than b, we're just going to subtract b again and get a - 2b and so on.
The mod is just a shortcut of doing those repeated steps in one step.

Here's the full code, I'll finish this post over the next few days:
```python
def gcd(a, b, c):
    mi = min(a,b)
    ma = max(a,b)
    if ma % mi == 0:
        return c
    c.append([ma, mi, ma // mi, ma % mi])
    return gcd(a % b, b, c) if a > b else gcd(b % a, a, c)

def egcd(a, b):
    arr = gcd(a, b, [])[::-1]
    l = arr[0]
    s = [l[3], l[0], 1, l[1], -l[2]]
    for aa in arr[1:]:
        s[3] = aa[0]
        s[2] -= aa[2] * s[4]
        s[2], s[4] = s[4], s[2]
        s[1], s[3] = s[3], s[1]
    return f"{s[0]} = {s[1]} * {s[2]} + {s[3]} * {s[4]}"
```
