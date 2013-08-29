    #! /usr/bin/env perl6

    use v6;

    constant UP = -1;
    constant DOWN = 1;

    sub score (*@wires) {
        gather for @wires -> $w {
            take [$w, benefit($w, DOWN) + benefit($w+1, UP)];
        }
    }

    sub benefit ($wire, $direction) {
        given @*current[$wire] {
            when 0 { return -1 }
            default { return $_ * $direction }
        }
    }

    sub swap (*@wires) {
        my @swapped = False,False,False,False,False,False,False,False,False;
        for @wires -> ($wire, $weight) {
            if $weight > 0 && !(@swapped[$wire] || @swapped[$wire+1]) {
                @swapped[$wire, $wire+1] = (True, True);
                @*current[$wire, $wire+1] = @*current[$wire+1, $wire];
                @*current[$wire] += DOWN;
                @*current[$wire+1] += UP;

                my $wire-ind = $wire * 2;
                @*wiring[$wire-ind] ~= '  ';
                @*wiring[$wire-ind+1] ~= '\\/';
                @*wiring[$wire-ind+2] ~= '/\\';
                @*wiring[$wire-ind+3] ~= '  ';
            }
        }

        for 0..9 -> $i {
            if !@swapped[$i] {
                @*wiring[$i*2] ~= '__';
                @*wiring[$i*2+1] ~= '  ';
            }
        }
    }

    sub wire (@*wanted) {
        my @*wiring;
        my @*current;
        for 0..9 -> $i {
            @*current[@*wanted[$i]] = $i - @*wanted[$i];
            @*wiring[$i*2] ~= "$i _";
            @*wiring[$i*2+1] ~= "   ";
        }
        while all(@*current) !~~ 0 {
            0..8 ==> score() ==> sort(-*[1]) ==> swap;
        }
        for 0..9 -> $i {
            @*wiring[$i*2] ~= "_ @*wanted[$i]";
            @*wiring[$i*2+1] ~= "   ";
        }
        return @*wiring;
    }

    sub invalid {
        say "Invalid input.";
        exit;
    }

    multi MAIN ($wiring) {
        my @wanted;
        {
            @wanted = $wiring.split('').map: { .Int };
            CATCH {
                default { invalid }
            }
        }
        if +@wanted !~~ 10 || @wanted.clone.sort !~~ 0..9 {
            invalid;
        }
        eager wire(@wanted)[0..18].map: { .say }
    }

## Correctness

This program is correct. It prefers to do flips as far left as possible.

It doesn't win the sub-award for elegance, however. Here's a case where the
solution is one column longer than it needs to be.

    $ perl6 bruce 6319845270
    0 _  __  ____  _________ 6
       \/  \/    \/           
    1 _/\  /\__  /\_________ 3
         \/    \/             
    2 _  /\__  /\___________ 1
       \/    \/               
    3 _/\__  /\  __  _______ 9
           \/  \/  \/         
    4 ___  /\  /\  /\_______ 8
         \/  \/  \/           
    5 _  /\  /\  /\  _______ 4
       \/  \/  \/  \/         
    6 _/\  /\  /\  /\  _____ 5
         \/  \/  \/  \/       
    7 _  /\  /\__/\  /\  ___ 2
       \/  \/      \/  \/     
    8 _/\  /\______/\__/\  _ 7
         \/              \/   
    9 ___/\______________/\_ 0

As the `edgar`, `moritz` and `vvorr` solutions show, there are solutions where
the 0 wire doesn't have to pause for one column, but can go all-diagonal all
the way.

## Consistency

The code is clean, clear, and consistent.

## Clarity of intent

The `@*wanted` parameter to `wire` is dynamic for no particular reason. Maybe a
confusion with slurpy arrays?

Speaking of dynamic variables, the reason the variables `@*wiring` and
`@*current` are dynamic in `wire` is that they're needed in `swap`. A slightly
less drastic (and more encapsulating) solution would have been to nest the
first three subs within `wire`, since that's where they're called anyway.

## Idiomatic use of Perl 6

There's a line in the middle of this algorithm that deserves highlighting:

    0..8 ==> score() ==> sort(-*[1]) ==> swap;

Cute use of the feed operator. The last step is called exclusively for its side
effects, but that feels fine considering it's put last in the chain.

However, at

    my @swapped = False,False,False,False,False,False,False,False,False;

we have a missed opportunity to do the slightly more Perlish list repetition:

    my @swapped = False xx 9;

Also, slight nit, but `+@wanted !~~ 10` over at line 77 suffers from some
insufficient specificity. Better written as a numeric comparison: `@wanted !=
10`.

## Brevity

Program clocks in at 81 lines, which is definitely OK.
