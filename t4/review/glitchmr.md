    #!/usr/bin/env perl6
    # The hacky way to have error checking
    my @points = gather for lines() {
        my $code = $_;
        take [
            try {
                CATCH { say qq[Unrecognized line: "$code"]; exit };
                .eval
            }
        ];
    };
    my ($x-bound, $y-bound, $z-bound) = (^3).map: {$(@points.map(*[$^a]).minmax)};
    my %world;
    my %checked;

    my $saved-water = 0;

    # This is recursive subroutine.
    sub waterfall($x, $y, $z) {
        # If cached, that means the field was already checked
        return True if %checked{$x}{$y}{$z};
        return False if $y < $y-bound.min || %world{$x}{$y}{$z};
        waterfall $x, $y - 1, $z;
        if %world{$x}{$y - 1}{$z} {
            %checked{$x}{$y}{$z} = 1;
            my $count = 0;
            # Perhaps it could be checked better, but I think this is good
            # enough check.
            $count += !waterfall $x - 1, $y, $z;
            $count += !waterfall $x + 1, $y, $z;
            $count += !waterfall $x, $y, $z - 1;
            $count += !waterfall $x, $y, $z + 1;
            if $count == 4 {
                $saved-water++;
                # Change water into wall
                %world{$x}{$y}{$z} = True;
                return False;
            }
        }
        return True;
    }

    for @points {
        # I really would like to know about more idiomatic solution, if possible.
        %world{$_[0]}{$_[1]}{$_[2]} = True;
    }

    # For every field on the map, water should fall
    for @$x-bound X @$z-bound -> $x, $z {
        waterfall $x, $y-bound.max + 1, $z;
    }
        
    say $saved-water;

## Correctness

This program is not correct.

* It gives 0 water for some containers that can contain some water.
* It sometimes fills only a small vessel when a surrounding bigger vessel could
  also be filled.

Furthermore, and I think this must go under "correctness", the script cleverly
avoids doing any kind of parsing by sending the input directly to `.eval`.
(These days spelled `.EVAL`.)

While the solution does get to the point quickly and will serve us well here,
it's also a gaping injection attack and wouldn't do well in any real-world
programming. Because this is just a concept, I will let it slide with a stern
warning.

## Consistency

Not much to comment on.

## Clarity of intent

This comment:

    # This is recursive subroutine.

Not sure why that needs to be pointed out. Would prefer a description of what
the subroutine actually is *for*, instead of how it does its thing.

Unless I'm much mistaken, this expression (from line 5)

    (^3).map: {$(@points.map(*[$^a]).minmax)}

is better written

    (^3).map: { $(@points[$^a].minimax) }

I'm not really sure why `$x-bound`, `$y-bound` and `$z-bound` weren't declared
as `@`-sigil variables. They're used as arrays throughout.

Maybe the error on line 7 should have been printied to `$*ERR` with `note`
rather than `say`. But that's a minor thing.

## Idiomatic use of Perl 6

This comment:

    for @points {
        # I really would like to know about more idiomatic solution, if possible.
        %world{$_[0]}{$_[1]}{$_[2]} = True;
    }

I live to serve. See bruce's custom `postcircumfix` for one possible solution.
Barring that, one nice way to make the above read better is to use nested
signatures:

    for @points -> [$x, $y, $z] {
        %world{$x}{$y}{$z} = True;
    }

Destructuring &mdash; we likes it, precious.

## Brevity

Incredibly short. Then again, the program gets most answers wrong...
