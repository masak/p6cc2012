    #! /usr/bin/env perl6

    use v6;

    class Vector {
        has $.x;
        has $.y;
        has $.z;

        method new (*@c) {
            return self.bless(*, :x(@c[0]), :y(@c[1]), :z(@c[2]));
        }

        method coordinates () {
            return $.x, $.y, $.z;
        }
    }

    multi infix:<-> (Vector $a, Vector $b) {
        return Vector.new($a.coordinates >>-<< $b.coordinates);
    }

    multi infix:<+> (Vector $a, Vector $b) {
        return Vector.new($a.coordinates >>+<< $b.coordinates);
    }

    class Cube {
        has Vector $.location handles ['x', 'y', 'z'];
        has $.has-water is rw = False;
        has $.is-rock is rw = False;
        has $.drainage-level is rw = Inf;
        has $.rained-on is rw = False;

        method above () {
            return $*world[$.location + Vector.new(0,1,0)];
        }

        method below () {
            return $*world[$.location - Vector.new(0,1,0)];
        }

        method beside () {
            gather for (1, -1) -> $diff {
                take $*world[$.location + Vector.new($diff, 0, 0)];
                take $*world[$.location + Vector.new(0, 0, $diff)];
            }
        }

        method drained() {
            return $.drainage-level ~~ $.y;
        }
    }

    class World {
        has @.space is rw;
        has @.dimensions;

        method new (@size) {
            # Create cube space with a bounding box of falling water
            my @world;
            for 0..@size[0] X 0..@size[1] X 0.. @size[2] -> $x, $y, $z {
                my $cube = Cube.new(:location(Vector.new($x, $y, $z)));
                if any(0 <<~~<< ($x, $y, $z)) || any(@size >>~~<< ($x, $y, $z)) {
                    $cube.drainage-level = $y;
                }
                @world[$x][$y][$z] = $cube;
            }

            return self.bless(*, :space(@world), :dimensions(@size));
        }

        multi method postcircumfix:<[ ]> (Vector $v) {
            return @.space[$v.x][$v.y][$v.z];
        }
    }

    grammar Input {
        token TOP { ^ [ <vector> | <malformed> ]* $ }
        rule vector { \( <number> <number> <number> \) \n? }
        rule malformed { (\N+) \n? }
        rule number { <.ws> ( \-? \d+ ) \,? <.ws> }
    }

    class Actions {
        has @min = Inf, Inf, Inf;
        has @max = -Inf, -Inf, -Inf;

        method TOP ($/) {
            @min = @min >>-<< (1,1,1);
            @max = @max >>-<< @min;
            @max = @max >>+<< (1,1,1);

            # The world origin is shifted to 0,0,0 so it can be stored in a 3-dimensional array
            my $world = World.new(@max);
            for $<vector>>>.ast >>->> Vector.new(@min) -> $loc {
                $world[$loc].is-rock = True;
            }

            make $world;
        }

        method vector ($/) {
            my @coord = $<number>.map({ $_[0].Str.Int });
            @min = @min >>min<< @coord;
            @max = @max >>max<< @coord;
            make Vector.new(@coord);
        }

        method malformed ($/) {
            die 'Unrecognized line: "' ~ $[0] ~ '"';
        }
    }

    sub fill-up (Cube $dest, Cube $source) {
        @*actions[$dest.y].push(sub {
            if $dest.is-rock {
                my $below-source = $source.below;
                if $dest.y ~~ $source.y && ($below-source.is-rock || $below-source.has-water) {
                    fill-up($source.above, $source);
                }
                return;
            }
            if $dest.has-water {
                return;
            }
            if $dest.drainage-level < $source.drainage-level {
                drain($source, $dest);
            }
            if $dest.drained {
                return;
            }
            rain-on($dest.below, $dest);
            eager $dest.beside.map: { fill-up($_, $dest) }
            if !$dest.has-water {
                $dest.has-water = True;
                $*volume++;
            }
        });
    }

    sub drain (Cube $dest, Cube $source) {
        @*actions[$dest.y].push(sub {
            if $dest.is-rock {
                return;
            }
            if $dest.drained {
                if !$source.drained {
                    drain($source, $dest);
                }
                return;
            }
            my $target-level;
            if $dest.y > $source.y && $source.drained {
                $target-level = min $dest.drainage-level, $source.drainage-level + 1;
            }
            else {
                $target-level = min $dest.drainage-level, $source.drainage-level;
            }
            if $target-level !~~ $dest.drainage-level {
                drain($dest.above, $dest);
                my @sides = $dest.beside;
                if $dest.below.is-rock {
                    eager @sides.map: { drain($_, $dest) }
                }
                else {
                    drain($dest.below, $dest);
                }
                $dest.drainage-level = $target-level;
                if $dest.drained {
                    if $dest.has-water {
                        $dest.has-water = False;
                        $*volume--;
                    }
                }
            }
        });
    }

    sub rain-on (Cube $dest, Cube $source) {
        @*actions[$dest.y].push(sub {
            if $dest.is-rock {
                if $dest.y ~~ $source.y {
                    fill-up($source, $dest);
                }
                else {
                    eager $source.beside.map: { rain-on($_, $source) }
                }
                return;
            }
            if $dest.drained {
                drain($source, $dest);
                return;
            }
            if !$dest.rained-on {
                $dest.rained-on = True;
                rain-on($dest.below, $dest);
                if $dest.drainage-level < $source.drainage-level {
                    drain($source, $dest);
                }
            }
        });
    }

    sub simulate-rainfall () {
        my @*actions;

        # Seed the clouds
        for 1..$*world.dimensions[0] X $*world.dimensions[1] X 1..$*world.dimensions[2] -> $x, $y, $z {
            my $cube = $*world.space[$x][$y][$z];
            rain-on($cube.below, $cube);
        }

        my $action = True;
        repeat {
            $action = False;
            for @*actions -> $level {
                if $level.defined && +$level {
                    $action = $level.pop;
                    $action();
                    last;
                }
            }
        } while $action;
    }

    sub calculate ($cubes, $verbose) {
        my $*volume = 0;
        {
            my $*world = Input.parse($cubes, :actions(Actions.new)).ast;
            simulate-rainfall;
            if $verbose {
                print-world($*world.space, $*world.dimensions);
            }
            CATCH {
                default { $*volume = ~$_ }
            }
        }
        return $*volume;
    }

    sub print-world (@space, @size) {
        use Term::ANSIColor;
        for (0..@size[1]).reverse -> $y {
            for 0..@size[2] -> $z {
                for 0..@size[0] -> $x {
                    my $c = @space[$x][$y][$z];
                    if $c.is-rock {
                        print colored(' ', 'on_red');
                    }
                    else {
                        my $fg = $c.rained-on ?? 'red' !! 'white';
                        my $bg = $c.has-water ?? ' on_blue' !! '';
                        print colored(~($c.drainage-level ~~ Inf ?? 'I' !! $c.drainage-level), "$fg$bg");
                    }
                }
                print ' ';
            }
            say;
        }
    }

    multi MAIN {
        say calculate($*IN.slurp, False);
    }

    multi MAIN ($file, Bool :$v?) {
        #This main is useful for debugging with Rakudo::Debugger because the debugger takes all stdin as its input
        say calculate(slurp($file).split("===\n")[1], $v);
    }

    multi MAIN(Bool :$test!, Bool :$v?) {
        use Test;
        use File::Find;
        for find(dir => 't').grep(/txt$/) -> $test {
            my ($volume, $cubes) = slurp($test).split("===\n");
            is(calculate($cubes, $v), $volume.chomp, $test);
        }
        done();
    }

## Correctness

The code is not correct -- it gets 0 water for some containers that can actually contain water.

Also, this line:

    die 'Unrecognized line: "' ~ $[0] ~ '"';

Might have compiled correctly at the point the program was submitted, but it
doesn't anymore. It should be `$/[0]`.

## Consistency

I found no particular inconsistencies.

## Clarity of intent

Arguably, a grammar rule called `number` shouldn't parse an optional comma.

In `return $.drainage-level ~~ $.y;`, the smartmatch is not conveying the fact
that `$.y` is always a number. `return $.drainage-level == $.y;` would have
been clearer.

## Idiomatic use of Perl 6

A `Vector` class is created with the primary purpose of declaring operators on
it. And yes, this is (in my opinion) exactly what we can extend `infix:<+>` and
`infix:<->` for &mdash; because vectors are "numbers" in some abstract sense,
and follow arithmetic laws.

A missed opportunity to declare a circumfix for vector literals, though.

    sub circumfix:<vec[ ]>(@c) { Vector.new(|@c) }

*Brilliant* re-declaration of `postcircumfix:<[ ]>` in `World`! I've never seen
this particular pattern before. Discovering new patterns through the contest
makes me very happy! Bravo!

## Brevity

The program reads nicely, like a story. Not too long, not too short.
