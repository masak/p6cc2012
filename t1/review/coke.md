    #!/usr/bin/env perl6

    enum Person <knave knight>;

    grammar Smul {
        rule TOP {
            <is-a> | <same> | <different> | <invalid>
        }

        rule is-a {
            <speaker=.person> ':' <target=.person> 'is a ' $<thing>=( 'knight' | 'knave') '.'
        }

        rule same {
            <speaker=.person> ':' <target=.person>+ % ' and '  'are of the same type.'
        }
     
        rule different {
            <speaker=.person> ':' <target=.person>+ % ' and ' 'are of different types.'
        }

        token person { <.alpha>+ }

        rule invalid {
            <.person> ':' {
                # looks like an utterance, but isn't really.
                die("Unrecognized utterance '" ~ $/.orig.Str.substr($/.to) ~ "'")
            } | {
                # garbage.
                die("Lines must be on the form '<name>: <utterance>'");
            }
        }
    }

    class Actions {
        has %.mentions;
        has %.knights;
        has %.knaves;
        has %.sames;
        has %.diffs;

        method is-a($/) {
            if $/<thing> eq "knight" {
                %.knights{$/<speaker>} = %.mentions{$/<speaker>} = ~$/<target>;
            } else {
                %.knaves{$/<speaker>} = %.mentions{$/<speaker>} = ~$/<target>;
            }
        }

        method same($/) {
            for $/<target> {
                %.mentions{$/<speaker>} = ~$_;
            }
            %.sames{$/<speaker>} = map {~$_}, $/<target>;
        }

        method different($/) {
            for $/<target> {
                %.mentions{$/<speaker>} = ~$_;
            }
            %.diffs{$/<speaker>} = map {~$_}, $/<target>;
        }

        method validate {
            # are all the mentions in the speakers?
            my $silent = (set(%.mentions.values) (-) set(%.mentions.keys));
            # ?? eventually emit multiple issues.
            if ?$silent {
                $silent = $silent[0];
                my $talker = %.mentions.invert.hash{$silent};
                return "$talker mentions $silent but $silent doesn't say anything.";
            }
        }

        method solve {
            # Anything to solve?
            return "" unless %.mentions;

            # Can we actually solve this?
            # We know all the statements. We know all the people who spoke.

            # Create a matrix of all possible solutions. 

            # ?? Surely there is a more sixish way to construct this object.
            my @worlds;
            my @mentions = %.mentions.keys;

            my $first = @mentions.shift;
            @worlds.push({$first => knight});
            @worlds.push({$first => knave});

            while (@mentions) {
                my $next = @mentions.shift;
                my @new;
                my $pos = 0;
                while (@worlds) {
                    my %world = @worlds.shift;
                    my %world2 = %world.clone;
                    %world{$next} = knight;
                    @new[$pos++] = %world;
                    %world2{$next} = knave;
                    @new[$pos++] = %world2;
                } 
                @worlds = @new.clone;
            }

            # See if the statements make sense with all the remaining solutions. 

            for %.knights.kv -> $speaker, $target {
                if ($speaker ne $target) {
                    # if someone says they themselves are a knight, they could be either, skip them
                    my @new; my $pos = 0;
                    for @worlds -> %world {
                        #     SPEAKER
                        # TAR KNI KNA
                        # KNI  Y   N
                        # KNA  N   Y

                        my $speakerType = %world{$speaker};
                        my $targetType  = %world{$target};
                        if $speakerType == $targetType {
                            @new[$pos++] = %world;
                        }
                    }
                    @worlds = @new;
                }
            }

            for %.knaves.kv -> $speaker, $target {
                if ($speaker eq $target) {
                    # It's impossible for either type to say this about themselves.
                    return "No solutions.\n";
                } else {
                    my @new; my $pos = 0;
                    for @worlds -> %world {
                        #     SPEAKER
                        # TAR KNI KNA
                        # KNI  N   Y
                        # KNA  Y   N

                        my $speakerType = %world{$speaker};
                        my $targetType  = %world{$target};
                        if $speakerType != $targetType {
                            @new[$pos++] = %world;
                        }
                    }
                    @worlds = @new;
                }
            }

            for %.sames.kv -> $speaker, @targets {
                my @new; my $pos = 0;
                for @worlds -> %world {
                    #     SPEAKER
                    # SAM KNI KNA
                    #  Y   Y   N
                    #  N   N   Y

                    my $same = [eq] map {%world{$_}}, @targets;

                    my $speakerType = %world{$speaker};
                    if $speakerType == $same {
                        @new[$pos++] = %world;
                    }
                }
                @worlds = @new;
            }

            for %.diffs.kv -> $speaker, @targets {
                my @new; my $pos = 0;
                for @worlds -> %world {
                    #     SPEAKER
                    # SAM KNI KNA
                    #  Y   N   Y
                    #  N   Y   N

                    my $same = [eq] map {%world{$_}}, @targets;

                    my $speakerType = %world{$speaker};
                    if $speakerType != $same {
                        @new[$pos++] = %world;
                    }
                }
                @worlds = @new;
            }

            # If there are none left, say so.
            unless @worlds {
                return "No solutions.";
            }
            # multiple? list them all.
            if +@worlds > 1 {
                return "Many solutions:\n" ~ join("", do { gather
                    for @worlds -> %world {
                        take join("; " , map { $_ ~ ': ' ~ %world{$_} },  %world.keys) ~ "\n";
                    } });
            }
            # one? say that one.
            return join("; " , map { $_ ~ ': ' ~ @worlds[0]{$_} },  @worlds[0].keys) ~ "\n";
        }
    }

    my $actions = Actions.new();

    # Parse each line, taking the appropriate action or output an error.
    for lines() -> $line {
        try {
            Smul.parse($line, :actions($actions));
            CATCH {
                when * {
                    say ~$_;
                    exit 1;
                }
            }
        }
    }

    my $reason = $actions.validate;
    if $reason {
        say $reason;
        exit 1;
    }

    print $actions.solve;

## Correctness

The solution appears correct.

## Consistency

The code is admirably straightforward and consistent.

## Clarity of intent

This solution has the advantage that it reads well, from top to bottom. The
comments with the tables help explain what's going on.

## Algorithmic efficiency

`solve` bifurcates all possible worlds, and then iterates it several times. So
we're clearly exponential.

## Idiomatic use of Perl 6

Aww, a missed opportunity to do `:actions($actions)` as `:$actions` instead.
I'm sorry, I just really like those.

## Brevity

Not so short, but not wastefully long either. Takes its time getting there, but
for decent reasons.
