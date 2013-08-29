    use v6;

    # solution to
    # "Arrange wire crossings to rearrange wires."
    # http://github.com/masak/p6cc2012/blob/master/t3
    #
    # Author: Kai Sengpiel (pKai)
    #
    # Outline of solution:
    # Bubble sort the unique_sequence_of_chars with the additional
    # constraint, that any number/pin/wire can only be
    # moved once during each loop iteration.
    #

    # We accept any unique sequence of characters
    # including the 10 digit permutations from the task examples.
    # Variables for book-keeping these chars:
    my @seq;
    my %idx_of_char;
    my @idxseq;
    my %char_of_idx;

    # Main data array (AoA)
    my @wiring;

    constant @icon =
        [ '\\/', '  ', '  ' ],
        [ '/\\', '__', '  ' ];

    sub display-wiring() {
        # We will never create columns without any crossings,
        # except in the case when we already entered with a sorted sequence.
        # Special case to concider:
        my $void = @wiring.elems == 2 && [?&](@wiring[0].list Z== @wiring[1].list);

        @wiring = transpose(@wiring);
        # @wiring now is a AoA which resembles the printed output in orientation
        for @wiring -> @row {
            my @difference_sequence_of_row =
                $void ?? Nil !! @row[1..@row.end] Z- @row;
            my @once_incremented_signum_sequence_for_differences_in_row
                = ( @difference_sequence_of_row.map({1+(!$_ ?? $_ !! $_ div $_.abs)}) );
            my @num = 
                [ '   '       , '   ' ],
                [ "%char_of_idx{@row[0]} _" , "_ %char_of_idx{@row[*-1]}" ];
            my @d := @once_incremented_signum_sequence_for_differences_in_row;
            my $line= -> $i { @num[$i][0] ~ @d.map({ @icon[$i][$_] }).join('') ~ @num[$i][1] };
            say $line($_) for +!?@row[0] ..1; # Is coercion Bool -> Int well-defined?
        }
    }

    sub compute-sorting() {
        @wiring = @idxseq.item;
        my @c = @wiring[0].list;   # why do I need .list? (Also at previous sub with Z==.)
        my Bool $same;
        repeat {
            $same = True;
            loop (my $i = 1; $i < @seq.elems; ++$i) {
                next if @c[$i-1] < @c[$i];
                @c[$i-1, $i] = @c[$i, $i-1];
                ++$i; # skip next, b/c we moved there.
                $same = False;
            }
            unshift @wiring, [@c];
        } until $same;
        @wiring.shift if @wiring.elems > 2;
    }

    # sub scraped from http://rosettacode.org/wiki/Matrix_transposition#Perl_6 
    sub transpose (@m) { 
        my @t;
        for ^@m X ^@m[0] -> $x, $y { @t[$y][$x] = @m[$x][$y] }
        return @t;
    }

    # Accept any unique sequence of chars
    sub MAIN($unique_sequence_of_chars where { $^p.chars == $^p.comb(/./).uniq.elems }) {
        @seq = $unique_sequence_of_chars.comb(/./);
        %idx_of_char = @seq.sort Z=> ^@seq;
        @idxseq = %idx_of_char{@seq};
        %char_of_idx = %idx_of_char.invert;

        compute-sorting();
        display-wiring();
    }

    # vim: nu ai et sts=4 sw=4 ts=4 ft=perl6

## Correctness

Yes, correct. Nice. Solution prefers to flip towards the right.

It doesn't always generate optimal solutions, however. Here's an example of a
solution that's one column wider than it has to.

    $ perl6 pkai 6319845270
    0 _  ____________  __  _ 6
       \/            \/  \/   
    1 _/\  __  ______/\  /\_ 3
         \/  \/        \/     
    2 ___/\  /\  __  __/\___ 1
           \/  \/  \/         
    3 _____/\  /\  /\  __  _ 9
             \/  \/  \/  \/   
    4 _______/\  /\  /\  /\_ 8
               \/  \/  \/     
    5 _______  /\  /\  /\___ 4
             \/  \/  \/       
    6 _______/\  /\  /\__  _ 5
               \/  \/    \/   
    7 _______  /\  /\__  /\_ 2
             \/  \/    \/     
    8 _______/\  /\____/\  _ 7
               \/        \/   
    9 _________/\________/\_ 0

As the `edgar`, `moritz` and `vvorr` solutions show, there are solutions where
the 0 wire doesn't have to pause, but can go all-diagonal all the way.

## Consistency

It seems to me that the only reason `transpose` was needed is to convert
between the format `compute-sorting` generates and the format `display-wiring`
needs. Instead of converting, they could have just agreed on one common format
and saved some code.

## Clarity of intent

When a variable name is so long the first thing you do with the variable is
rebind it to a shorter name, maybe take that as a signal to look for a shorter
name? `:-)` `@once_incremented_signum_sequence_for_differences_in_row` is
descriptive to the max, but so is `@signs_incr`.

Re the line

    say $line($_) for +!?@row[0] ..1; # Is coercion Bool -> Int well-defined?

Yes, it is well-defined, but relying on it usually isn't the clearest way to
convey an idea. In this case, this could have worked just as well.

    say $line($_) for @row[0] ?? 0..1 !! 1..1;

## Idiomatic use of Perl 6

`.comb(/./)` could also be written as `.comb` &mdash; shorter but admittedly
also more implicit.

By the way, here's a cool alternative way to implement the `transpose` function:

    sub transpose(@m) { ([Z] @a).tree }

Yes, that's right. Zipping and transposition are the same operation.

## Brevity

A fairly short solution.
