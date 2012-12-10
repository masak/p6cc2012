## Tell knights from knaves based on what they say

On the mythical island of Smul, people suffer from a rare genetic disorder that
make them either tell the truth all the time, or lie all the time. These are
the only two types of people on the island, known as knights and knaves,
respectively.

Write a program that takes as input a number of utterances by islanders, and
outputs for each person whether that person is a knight or a knave. If there is
no possible assignment that works, the program should report that no solution
exists. In the case of multiple solutions, the program should report every
possible solution.

The islanders can make four different classes of utterances:

    X is a knight.
    X is a knave.
    X and Y are of the same type.
    X and Y are of different types.

(Here, X and Y are used as metavariables, of course, and can in fact be any
name of an islander.)

Islanders can refer to each other. The same islander can make several
utterances. If an islander mentions another islander that doesn't say anything,
your program should consider the entire input to be erroneous.

Here are a few examples:

    A: A is a knight.

Both a knight and a knave would assert the same thing. So this input has two
solutions.

    B: B is a knave.

Neither a knight or a knave would ever say this about themselves. So this input
allows no solution.

    C: C and D are of the same type.
    D: D and C are of different types.

Here, the two islanders are contradicting each other, so one of them must be a
knight and the other a knave. But this is exactly what D is saying, so D is the
knight. One solution.
