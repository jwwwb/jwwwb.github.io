---
layout: post
title:  "Error Checking the Chess Bot."
date:   2016-07-17 23:44:44 +0200
categories: general
---
Once I got my [chess engine up and running rudimentarily]({% post_url 2016-07-14-chessbot %}) it was time to error check. For that purpose I used [perft][perft-wiki], basically the number of possible moves from a given position on the chessboard, for a given number of half-moves or plies (ie, perft from the start for 2 plies is 20 moves for white * 20 possible responses for black = 400). There are correct numbers for a variety of depths to be found on the internet, so you can simply compare the possibilities from your engine and if they differ, you have an error somewhere. I found a lot of useful information on [Albert Bertilsson's site][albert-page], who implemented a chess engine himself, and had the following advice to offer:

> **Warning:**
> If you like programming and solving mind games and puzzles **do not** start writing your own chess engine. I was tricked into chess programming in late 2002 and has never found any other programming to be as challenging and funny. Chess programming is like a discease without a cure, I've tried other programming projects but none of them has given me the same intellectual satisfaction. 

Well, too late for that now, I fear. But apart from that word of warning, he also had some useful perft values for comparison. And his chess engine, Sharper, has the wonderful feature of a split perft test, which rather than simply compute all the possible move combinations to some depth n, instead computes the perft value for every legal move from the current position to depth n-1, which should then sum back up to the original perft value. By implementing this in my engine (and hoping that Sharper's values are correct) I was able to easily debug in which move branches led to the errors in my perft numbers. Unsurprisingly it was special moves like en-passant and castling where I found a few errors.

With this being fixed, it's now time to attack the search and evaluation. Though I have already read a bit about how to best tackle this, I think the best idea for me is to continue as I had originally planned with some basic pointwise evaluation of piece losses and captures and an exhaustive search. Primitive as this might be, it should at least generate a somewhat legitimate player, which will be nice to see in action, before I move onto more complex algorithms there. 


[albert-page]: http://www.albert.nu/programs/sharper/perft/
[perft-wiki]: https://chessprogramming.wikispaces.com/Perft+Results
