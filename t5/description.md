## Distribute weights in bags evenly

Write a program that takes a positive integer N as input, followed by zero or
more positive numbers representing weights. The output should be those same
weights, but distributed into N bags.

The distribution should be such that all the bags weigh about the same. Or, put
differently, you'll want to minimize on some measure (of your own choosing)
that goes to zero when the bags weigh the same.

There's a weak expectation that your program not take a very long time to run,
even as the number of bags and weights grows. In fact, it's more important for
the program to scale well on N and the number of weights than for it to get the
optimal distribution all of the time.

Example input may look like this:

    3
    1.0 2.0 3.0 4.0 5.0

In this case, there's a perfectly balanced distribution:

    (1.0 4.0) (2.0 3.0) (5.0)

You are expected to provide canonical output, where within each bag the weights
are listed in their original order, and the bags are listed according to the
original order of their first weight. (And empty bags at the end.)

Here's an other example:

    3
    1 2

Here, we can obviously not balance things out, so we do the best we can:

    (1) (2) ()
