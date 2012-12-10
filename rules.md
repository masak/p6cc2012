## How to win

* A randomly chosen person wins...

* ...unless people actually send in solutions.

* Then the choice is made from the "winning set" of people who submitted
  solutions to the most exercises. (Put differently, if you've submitted
  solutions to strictly fewer exercises than someone else, you're not in the
  winning set.)

* If there's a single contestant in the winning set, that contestant wins.

* Otherwise, the choice of winner is made by us (masak & moritz) based as
  much as possible on code quality.

Since "code quality" is a slightly subjective measure, let us provide a few
hints of what we'll be looking for:

* Correctness.
* Readability.
* Consistency.
* Clarity of intent.
* Algorithmic efficiency.
* Idiomatic use of Perl 6.
* Brevity.

In short, what we're looking for is top-quality code. That's how you win.

You're free to make use of your own judgment in the shaping of your
code. If you use code that doesn't originate with you, please indicate
so in your comments. There's a spectrum of "copying" here: implementing
an algorithm from somewhere is perfectly fine; sending in a copy of
someone else's solution isn't.

One final thing. Please be aware that, by entering the contest, you're not
guaranteed the prize. That is, we're not legally bound to hand out the prize
to you for any reasons *you* can think of. We might decide to hand out the
prize to everybody, nobody, or the *worst* coder. We won't, though. We'll
hand out the prize to the best coder. So just be best, and you'll be fine.
Make sure you read each problem description carefully.

## How to write a solution

You're free to code however you want, but for the purposes of easy comparison
between submissions, we have a few requests.

* Put your main Perl 6 script in a file called `code` in the respective
  task directory.

* Make sure this code passes all the tests in `t1/base-test`,
  `t2/base-test`, etc.

We've created the `base-test` files to ascertain a couple of basic
requirements about input and output. This is just to have some common basis
of comparison between the submissions.

You're certainly allowed to write and include your own tests in a t/ directory,
and to separate code into modules in a `lib/` directory, as long as the file
`t1/code`, etc., remains the main script. The `base-test` script will set
`PERL6LIB` so that modules in the `lib/` directory are found.

With everything else -- choice of algorithm, indentation, comments, naming --
you're completely free -- encouraged, even -- to use your own good judgment.

When you're ready to submit a solution, send the file(s) along to
cmasak@gmail.com. Put them in a `.zip` file if you want. You're free to send in
solutions individually or all at once. Don't wait until the last moment with
sending in solutions.

If you're one of those Git octocats, you might want to consider cloning this
repository, and then just building an archive out of your latest version:

    $ git archive -o p6cc-solutions-by-jrandomhacker.zip HEAD

Both Rakudo and Niecza are appropriate platforms for developing solutions for
the contest, and we're flexible as to under which version/release you've
written your code.

See the [compiler feature matrix](http://perl6.org/compilers/features) or ask
on `#perl6` for up-to-date info about compiler features.

We appreciate if you tell us which compiler version you're using, but in
a pinch we'll figure it out. `:-)` The `base-test` files respond to an
environment flag `PERL6EXECUTABLE` that you can set if your Perl 6 executable
is not named `perl6`. For Niecza, something like `mono run/Niecza.exe` ought to
work.

You're allowed to send in several solutions to the same problem, but only the
last one will count, and please don't send more than a total of 10. :-) All
solutions will be published after the conclusion of the contest. All are up for
review.

## How to have the appropriate amount of fun

First off, if you haven't entered the contest yet &mdash; do so! You've nothing
to lose.

Secondly, Perl 6 is a shoestring project but with very dedicated people who
are working toward a single vision. (Or, on the days when we have a head-ache,
double vision.) Part of this whole enterprise is making sure we're learning,
about programming, about languages, and about communities. In some loose sense,
that's what this contest is about too: learning new things, and enjoying the
experience.

So, enjoy!
