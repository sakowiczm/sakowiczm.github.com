--- 
layout: post
title: Sudoku Solver in C#
comments: true
date: 2012-01-02
---

Another breakable toy, my variation of Sudoku solver. I've created it without previously googling the topic , and I was quite surprised when latter I realized that most solutions out there use just dumb trial and error. I was also glad that I can re-invent backtracking algorithm ;) 
 
Ok. let's start, from having a Sudoku that we want to solve:

![Example Sudoku](/img/posts/2011/2011-01-02-sudoku_thumb.png){: .center-image .img-responsive }

After a little consideration I decided to solve it by elimination of possible values. Let's consider first block (by block I mean inner 3x3 cells squares), possible values for empty cells:

![](/img/posts/2011/2011-01-02-table1.png){: .center-image .img-responsive }

We can see that value 1 can be only in A1 cell so fill it in. Check remaining cells and no single value in a row or column. So we evaluate next block:

![](/img/posts/2011/2011-01-02-table2.png){: .center-image .img-responsive }

Now value 1 and 6 are only possible in cells B5 and B4 so we fill them in. Check remaining cells after reduction - no single possible values - we go to the next block. 
 
We cycle through all blocks until all cells are filled in (will happen only for very simple Sudoku) or number of empty cells is not changed after a cycle. In the second case, we filled what we could using simple elimination and it's time for more advanced solving techniques or guessing. This time I choose guessing, maybe later I'll implement something more original. So we take first empty cell pick its first possible value fill it in and use recursion and [backtrack algorithm] [1] to check if we can complete the puzzle. If not go back pick next possible value and try again.

That's it the whole working solution can be found on [github] [2]. To test the solution I've used online solver at [Sudoku Wiki] [3]. Beside of detail description of different techniques of solving Sudoku, it allows to import and export puzzle as string of numbers which is really helpful. 
Next step for me is to implement the same algorithm using less familiar languages. I think of Python, Ruby, JavaScript and Dart, F# and maybe Haskell.


[1]: http://en.wikipedia.org/wiki/Backtracking
[2]: https://github.com/sakowiczm/Sudoku-Solver-CSharp
[3]: http://www.sudokuwiki.org/sudoku.htm