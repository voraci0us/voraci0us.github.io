<h1>Extended Euclidean</h1>

<b>This is a draft for testing... need to finish writing this and figure out LaTeX.</b>

As the fall semester ends - and with it, Guha's cryptography class - I've been looking for ways to keep learning. I started working my way through a site called CryptoHacks (https://cryptohack.org/challenges/). One of the early challenges involves calculating modular inverses.

I could have done
```python
pow(a, -1, n)
```
But I figured this was as good a time as any to write my own mod inverse from scratch.
This post will contain a quick crash course in modular arithmetic + some light number theory - if you know how arithmetic works, you should be fine.

So what's a mod inverse?

Let's start with what "mod" means.
In computing, "mod" is an operation that returns the remainder of integer division.
It's usually represented with a percent sign "%".
So 14 mod 5 is 4, since if you try to split 14 objects into groups of 5, you'll have 4 left.
Modular arithmetic is widely used in cryptography. Partially because we can restrict large numbers to a resonable side, and partially because of some properties that make a modulus operation hard to undo (https://en.wikipedia.org/wiki/Discrete_logarithm).

More generally, in math "mod" is a congruence relation.
Congruence relations mean "these two things are equal under this condition."
For example, we can say
14 = 4 (mod 5)
Obviously 14 and 4 aren't equal, but *under the condition* mod 5 they are (since 14 mod 5 is 4 and 4 mod 5 is also 4).
There's a few rules for how we can work with congruence relations.
Unlike equalities, we can't just apply any operations to both sides and have the condition still hold true.
For example, what if we divide both sides of the congruence relation by 5?
2.8 = 0.8 (mod 5)
This is meaningless - we can't do integer division (which mod is based on) with decimal numbers.

However, adding to both sides is fine.
14 + 3 = 4 + 3 (mod 5)
17 = 7 (mod 5)
17 mod 5 = 2
7 mod 5 = 2
Additionally, subtracting from and multiplying by both sides is fine. This is left as an exercise to the reader.

This makes an interesting problem. What if we want to solve this congruence relation?
3x = 2 (mod 5)
We can't divide both sides by 3, so how can we solve for x?
If we try numbers, we can find that an x value of 4 works. 3 * 4 = 12 = 2 (mod 5)
How can we find this efficiently?

This is where the concept of the modular inverse comes in.
Let's say we have an integer a and a modulus n.
The mod inverse a' of a satisfies this congruence relation: a * a' = 1 (mod n)
The value of a' depends both on the value of a and n.
So an integer a has a different inverse for different moduli.
This inverse exists iff (if and only if) a and n are coprime (they do not share any factors). The proof for this is beyond the scope of this post but you can learn more at (https://en.wikipedia.org/wiki/Modular_multiplicative_inverse).

How does this help us solve the equation?
3x = 2 (mod 5)
Well, if we know that the mod inverse of 3 mod 5 is 2, then we can multiple both sides of the equation by 2:
3 * 2 * x = 2 * 2 (mod 5)
By the definition of mod inverse, a * a' = 1
x = 2 * 2 (mod 5) = 4

Hopefully this gives you an idea of why modular inverses are useful - they bascially replace the concept of division in modular arithmetic.

So how do we find them? The Extended Euclidean Algorithm of course!
The Euclidean algorithm is a method of finding the GCD (greatest common divisor) of two numbers.
It relies on the fact that if you have two numbers a and b with a > b, GCD(a, b) == GCD (a - b, b).
In other words, if we replace the larger number with the difference between the numbers, the GCD stays the same!
We can take this a step further and say GCD(a, b) == GCD (a % b, b).
If a - b is still greater than b, we're just going to subtract b again and get a - 2b and so on.
The mod is just a shortcut of doing those repeated steps in one step.
