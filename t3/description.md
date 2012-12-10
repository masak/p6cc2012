## Arrange wire crossings to rearrange wires

Ten wires are come in from the left. Ten wires leave to the right. Write a
program that, given a permutation of the digits 0..9, arranges crossings on a
grid that places the wires in the order designated by the permutation.

The output consists of a grid whose elements connect the left and right sides.
Each cell of the grid is either *empty* (in that it just lets the wires
through), or a *crossing* (in that it lets the wires trade places). Two
crossings can never be placed vertically adjacent to each other. (This is
equivalent to saying that the wires never split or merge, they only permute.)

Often, solutions require crossings to be spread out not just vertically but
also horizontally. It is considered elegant not to make the grid wider than it
has to be.

Examples:

    Input: 1032547698

    Output:

    0 _  _ 1
       \/
    1 _/\_ 0

    2 _  _ 3
       \/
    3 _/\_ 2

    4 _  _ 5
       \/
    5 _/\_ 4

    6 _  _ 7
       \/
    7 _/\_ 6

    8 _  _ 9
       \/
    9 _/\_ 8

(This permutation is simply flipping wires pairwise.)

    Input: 1234567890

    Output:

    0 _  _________________ 1
       \/
    1 _/\  _______________ 2
         \/
    2 ___/\  _____________ 3
           \/
    3 _____/\  ___________ 4
             \/
    4 _______/\  _________ 5
               \/
    5 _________/\  _______ 6
                 \/
    6 ___________/\  _____ 7
                   \/
    7 _____________/\  ___ 8
                     \/
    8 _______________/\  _ 9
                       \/
    9 _________________/\_ 0

(The simplest way to bubble 0 to the bottom.)

    Input: 5012367894

    0 _________  _ 5
               \/
    1 _______  /\_ 0
             \/
    2 _____  /\___ 1
           \/
    3 ___  /\_____ 2
         \/
    4 _  /\_______ 3
       \/
    5 _/\  _______ 6
         \/
    6 ___/\  _____ 7
           \/
    7 _____/\  ___ 8
             \/
    8 _______/\  _ 9
               \/
    9 _________/\_ 4

(The simplest way to bubble 4 and 5 simultaneously.)
