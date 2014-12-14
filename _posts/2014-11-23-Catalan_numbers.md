---
title: Catalan number
layout: post
---
How many ways are there to push and pop n elements into and out a stack? We marked E as push and L as pop, the question can be converted to: how many ways to represent a 2n length string with E and L that in any of its prefixes, numbers of E are no less than numbers of L. If a 2n string composed by E and L can represent a way to push and pop n elements, we call it a good string.

From a 2n string with n E and n L, find first L before which it there're same Es and Ls.  If we can find such an L, the string doesn't satisfy conditions above. We named such strings as bad string. 

For a bad string, if we flip it after the first L before which there're same Es and Ls, we got another 2n string composed by n+1 L and n-1 E.

For 2 different bad strings B1 and B2 with different flip position, the flipped strings of them shouldn't be same. Assume B1's flip position is p1, B2's flip position is p2 and p1 < p2(index based 0, and p1 must > 0), then the lenght p1 prefix of blipped strings B1' and B2' are keeping unflipped and should be same with the original string. If the original strings have a same length p1 prefix, they should both flip at p1 position. So the prefix with a length p1 of B1 and B2 are different.

For 2 different bad strings B1 and B2 with same flip position p, there must be some different before or after p. No matter where the difference is, the difference part will be clipped or not samely for the two strings. So they keep different afte flip. 

So different bad strings got different flipped strings with n+1 L and n-1 E.

Vice visa, for any two different strings that both have n+1 L and n-1 E, we flipped back after the first L that before which there're same Es and Ls and get a string with same Es and Ls. Since there're more Ls than Es, we can always find such an L. We can prove similarily that different original strings will get different flipped strings.

So there's a 1-1 mapping from bad strings with n Es and n Ls to strings with n+1 L and n-1 E. So there're C(2n, n-1) bad strings.
The good string number is: C(2n, n) - C(2n, n-1) = C(2n, n) / (n+1). This is the famous [Catalan number](http://en.wikipedia.org/wiki/Catalan_number).

Cat(n) = C(2n, n)/(n+1).

Let's try to get Cat(n) in another way: suppose before we popped k elements out before popping the first element out, for k = 0..n-1, we have

Cat(n) = sum(Cat(k)Cat(n-1-k)) (k=0...n-1)

Example 1: Ways to generate valid parenthesis sequences. Consider left parenthesis as push and right parenthesis as pop.

Example 2: Add parenthesis to expression of N matrix multiply. P(1) = 1, P(2) = P(1) * P(1), P(3) = P(1) * P(2) + P(2) * p(1), P(n) = Cat(n-1)

Example 3: How many different shapes of a n nodes binary tree. T(0) = 1, T(1) = 1, T(2) = T(0) * T(1) + T(1) * T(0) (left from 0 to n-1, right from n-1 to 0), T(n) = Cat(n)

Example 4: 2n points on a circle, how many ways to conect those points to n segments without crossing each other. F(0) = 1, F(1) = 1, F(2) = F(0) * F(1) + F(1) * F(0), F(n) = F(0)* F(n-1) + F(1) * F(n-2) + ...  F(n-1) * F(0) = Cat(n) (Select a base point, connect it with a segment to the point that has even points between them, So in the two sides of the segment there're  2k points and 2n - 2 - 2k points...)

Example 5: Divide a convex polygon to triangles. T(2) = 1, T(3) = T(2)*T(2)=1, T(4) = T(2) * T(3) + T(3) * T(2)= 1, T(5) = T(2) * T(4) + T(3) * T(3) + T(4) * T(2) = Cat(n-2) (Select one side AB, select C from the other n-2 points to get a triangle, so there's a pologon of k points on the side of AC, k = 2 to n-1, on the side BC, there's a pologon of n+1-k points(C is the common points of two polygons), T(n)=sum(T(2)T(n-1)+...+T(n-1)T(2))



