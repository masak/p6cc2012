    use v6;

    # begin from the back, and calculate the transpositions
    # needed to to arrive at 0..9;
    sub next-transpositions(@perm) {
        my @displacement = @perm         Z-  @perm.keys;
        my @score        = @displacement Z-  @displacement[1..*];
        my @idx-to-swap  = @score.pairs.grep(*.value > 0).sort(*.value)>>.key;
        my %seen;
        gather for @idx-to-swap.reverse {
            next if %seen{$_} || %seen{$_ + 1 };
            %seen{$_    } = 1;
            %seen{$_ + 1} = 1;
            take $_;
        }
    }

    sub apply-transpositions(@perm, @trans) {
        my %c = @perm.keys Z=> @perm.keys;
        %c{ (0, 1) X+ @trans } = (1, 0) X+ @trans;
        @perm[%c{@perm.keys}];
    }

    sub MAIN ($target = '5249801376') {
        my @initial = $target.comb>>.Int;

        my @perm = @initial;
        my @transpositions = gather while next-transpositions(@perm) -> @trans {
            take @trans.item;
            @perm = apply-transpositions(@perm, @trans);
        }

        # the rest is just for output
        my @is-trans = @transpositions.reverse.map: -> @trans {
            my @bitmap      = False xx 9;
            @bitmap[@trans] = True  xx *;
            @bitmap.item;
        }
        for ^10 -> $n {
            say join '', "$n _",
                @is-trans.map(-> @t {
                    @t[$n]             ?? Q'  ' !!
                    $n > 0 && @t[$n-1] ?? Q'/\' !! '__'
                }),
                "_ @initial[$n]";
            if $n < 9 {
                say join '', '   ', @is-trans.map( -> @t {
                        @t[$n] ?? Q'\/' !! '  '
                    });
            }
        }
    }

## Correctness

This program is correct.

It prefers to do its flips as far to the right as possible.

I have found no case in which the program falls short on the elegance subgoal,
either. Bravo.

## Consistency

Extremely consistent.

## Clarity of intent

Very clear. Notice how the `MAIN` routine is divided into three sections:
input, processing, output. The two subs before are simply helpers that
package a few operations into a single idea.

I get a slight headache when I look at the last loop; the one that outputs
stuff. But my complaint is not really with the program itself, it's more
like "why does outputting code tend to look so weird?" *Is* there a good
way to output array-like things, such that an average programmer would look
at the code and nod and say "yes, that looks about right"? This code somehow
doesn't feel declarative enough &mdash; it's 90% logic and 10% output. If you
know what I mean.

## Idiomatic use of Perl 6

A `while` loop with a `->`. Nice.

## Brevity

A pleasantly short solution. Given that it's also the elegantest, I'd say being
succinct and to the point wins the day here.
