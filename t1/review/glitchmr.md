    #!/usr/bin/env perl6
    # This entry doesn't work in Niecza because Niecza doesn't have .tree
    # method for lists.

    my %conditions;
    my %types;
    my %mentions;
    enum Smul <Knight Knave>;

    # I don't know how to use grammars in Perl 6. I hope that for/when is
    # good enough replacement.
    for lines() {
        when / ^ (.*?) \: " " (.*?) ' is a ' [ (knight) | knave ] \. $ / {
            my ($name, $target, $knight) = $[0 .. 2];
            my $type = $knight ?? Knight !! Knave;

            # Push check of type
            push %conditions{$name}, { %types{$target} == $type };

            # This is needed for "error" checking. Technically, my code
            # could even guess type of Smul without making errors even if
            # he hasn't told anything, but masak wants error, so I've it.
            %mentions{$target} = $name;
        }
        when / ^ (.*?) \: " " (.*?) ' and ' (.*?) ' are of ' [('the same type')|'different types'] \. $ / {
            my ($name, $target1, $target2, $same) = $[0 .. 3];
            
            # Checks if both Smuls are the same type. If $same is not
            # defined, the result is reversed.
            push %conditions{$name}, { [==](%types{$target1, $target2}) == ?$same };

            # Needed for "error" checking. $name is duplicated two times.
            %mentions{$target1, $target2} = $name xx 2;
        }
        # Now for error checking. You would think that it's about
        # algorithms, but nope.
        when / ': ' (.*) / {
            say "Unrecognized utterance '$0'";
            exit;
        }
        default {
            # "on" is a mistake, but it's in test, so whatever
            say "Lines must be on the form '<name>: <utterance>'";
            exit;
        }
    }
    my @names = %conditions.keys;
    exit unless @names;

    # Now check what masak wants.
    for %mentions.keys {
        if !%conditions{$_} {
            say "%mentions{$_} mentions $_ but $_ doesn't say anything.";
            exit;
        }
    }

    # [X](['blah', 'bleh'] xx $numish) is brute-force idiom.
    my @solutions = gather for [X]([Knight, Knave] xx @names).tree -> $solution {
        my @solution = @($solution);
        %types{@names} = @solution;
        # Is every condition met.
        my $success = True;
        for %conditions.kv -> $name, @conditions {
            for @conditions -> &condition {
                # If type of name is knight, condition has to return true.
                # Otherwise, it has to return false. Parens needed because
                # Perl 6 would chain `==` operators.
                $success &&= (%types{$name} == Knight) == ?condition;
            }
        }
        # {%types} gives scalar version of scalar, just like in Perl 5.
        take {%types} if $success;
    };
    # Now check the number of solutions.
    unless @solutions {
        say 'No solutions.';
    }
    if @solutions > 1 {
        say 'Many solutions:';
    }
    .say for gather for @solutions -> %solution {
        my $result;
        for @names {
            $result ~= "; " if $result;
            $result ||= "";
            $result ~= "$_: " ~ (%solution{$_} == Knight ?? 'knight' !! 'knave');
        }
        take $result;
    }

## Correctness

The algorithm appears to be coorect.

## Consistency

No complaints about consistency. The naming is good throughout. Diligent use of
indentation.

## Clarity of intent

It's not at all clear to me why the contestant is storing nullary blocks (with
no closure semantics) into %conditions, all of which evaluate to booleans when
called, instead of just storing the booleans directly.

The comments are abundantly clear about not being fond of error checking.

## Algorithmic efficiency

This solution (quite explicitly) constructs and iterates through all possible
universes.

## Idiomatic use of Perl 6

Nice: `my ($name, $target, $knight) = $[0 .. 2];`

A bit of a "useless use of `gather`" at the end, where, instead of
`gather`/`take` and then `.say` on the resulting lazy list, you could have just
dropped the `gather/take` for the same effect (returning `$result` from the
block), or even done `say $result` in the loop directly.

## Brevity

The solution is wonderfully short.
