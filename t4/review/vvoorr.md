    use v6;

    # read the 'input' file
    my $input = slurp('input');
    unless $input {
        say 0;
        exit;
    }

    # parse the 'input' file
    # @input[0] == [xx, xx, xx]
    my @input <== map { [$_.comb( / \-? \d+ / ).map(+*)] } <== $input.lines;
    unless $input ~~ m/\d/ {
        $input = chomp $input;
        say "Unrecognized line: \"$input\"";
        exit;
    }

    # find out the boundary of the walls
    my @max = [Zmax] @input;
    my @min = [Zmin] @input;

    # shift the walls so that there is no negative indices
    my @boundary = @max >>-<< @min >>+>> 2;
    my @move = 0 <<-<< @min >>+>> 1;
    my @wall <== map {[@$_ >>+<< @move]} <== @input;

    # $space[$t][$u][$v] is a virtual space
    # $space[$t][$u][$v] = 1 is empty space
    # $space[$t][$u][$v] = 2 is wall
    # $space[$t][$u][$v] = 3 is not bounded by walls
    my $space = [ [ [ 1 xx @boundary[2]+1 ] xx @boundary[1]+1 ]  xx @boundary[0]+1 ];
    $space[@wall[$_][0]][@wall[$_][1]][@wall[$_][2]] = 2 for ^@wall.elems;

    $space[0][0][0] = 3;
    my $pointNotBounded = 1;
    f_search($space,(1,0,0),@boundary,$pointNotBounded);

    # print the number of trapped points
    # number of trapped points = number of points in space - number of walls - number of free points
    say ([*] @boundary >>+>>1) - @wall.elems - $pointNotBounded;









    sub f_search ($space,@curPosition,@boundary,$pointNotBounded is rw) {
        my ($a,$b,$c) = @curPosition;
        return if $space[$a][$b][$c] == one(2,3);
        
        # search the surrounding and see if any of them is 3, if so then turn itself into 3
        if (0 <= $b-1 <= @boundary[1]) && ($space[$a][$b-1][$c]) == 3 {
            $space[$a][$b][$c] = 3;
            $pointNotBounded++;
        } else {
            for ($a,$a,$a-1,$a+1) Z ($c-1,$c+1,$c,$c) -> $A,$C {
                if (0 <= $A <= @boundary[0]) && (0 <= $C <= @boundary[2]) && ($space[$A][$b][$C] == 3) {				
                    $space[$a][$b][$c] = 3;
                    $pointNotBounded++;
                    last;
                }			
            } 
        }
        
        return if $space[$a][$b][$c] != 3;
        
        if 0 <= $b+1 <= @boundary[1] {
            f_search($space,($a,$b+1,$c),@boundary,$pointNotBounded);
        } 
        for ($a,$a,$a-1,$a+1) Z ($c-1,$c+1,$c,$c) -> $A,$C {
            f_search($space,($A,$b,$C),@boundary,$pointNotBounded) if (0 <= $A <= @boundary[0]) && (0 <= $C <= @boundary[2]);
        } 		
    }

## Correctness

The program is not correct:

* It assumes that water can flow into containers which are covered or completely sealed.
* It assumes that the water level can rise above the lowest outlet.

The program makes a big point of searching for cells which are "bounded by
walls". I'm not so sure that's a metaphor that would get us all the way to a
correct algorithm. It's more about which cells are inside a container capable
of holding water without leaking it out.

## Consistency

This program likes camelCase for variables and underscore_separation for its
one subroutine. Whether that's an inconsistency or a sensible separation is a
matter of taste, I guess.

## Clarity of intent

If I were to describe the style of this program in one word, I'd call it
"APL-ish". `:-)` Some of the operations in the program fit that kind of
thinking very nicely, for example when you realize that

One tip: If you find yourself leaving a comment such as this one:

    # $space[$t][$u][$v] is a virtual space
    # $space[$t][$u][$v] = 1 is empty space
    # $space[$t][$u][$v] = 2 is wall
    # $space[$t][$u][$v] = 3 is not bounded by walls

...then consider very carefully whether you would not prefer to express that
comment as an enum in the code instead. That would eliminate many of the magic
constants spread around in the code, and also possibly invite some type safety.

Also, if you find yourself naming your variables in the main subroutine `$a`,
`$b`, `$c`, `$A`, and `$C`... stop for a while and consider whether there might
not be better names for some of these variables. What makes you want to call
that particular variable `$C`? More clues might make it clearer to the reader.

## Idiomatic use of Perl 6

I like the liberal use of reduction operators (`[Zmax]`), pipe operators
(`<==`), hyperoperators (`@max >>-<< @min >>+>> 2`) and chained comparisons (`0
<= $b-1 <= @boundary[1])`). That's certainly making use of Perl 6's
expressivity.

The subroutine keeps passing `$pointNotBounded` to itself as an `rw` variable.
The exact same effect could have been had by *not* passing it and just directly 
modifying the global. This reveals the deeper problem that the program
currently does its calculation by updating a global variable. Not necessarily a
big problem at this scale, but if the program were to grow, it would quickly
become one.

## Brevity

Admiringly short, recursive solution. Then again, it gets many of the cases
wrong.
