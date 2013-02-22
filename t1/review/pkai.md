    use v6;
    use Logic;

    # solution to
    # "Tell knights from knaves based on what they say."
    # http://github.com/masak/p6cc2012/blob/master/t1
    #
    # Author: Kai Sengpiel (pKai)
    #
    # Additional assumptions made for this solution:
    # * Names of Smul inhabitants match \w+ and are concidered case-sensitive.
    # * If any of the errors described in the test cases are encountered,
    #   we will report only the first and stop thereafter.
    #
    # Outline of solution:
    # All statements to concider will be transformed to
    # a minimal disjunctive normal form (boolean logic).
    # From this the final result can be deduced directly.
    #


    sub parse-stdin-to-Logic() {
        my @lines = $*IN.slurp.comb(/<-[\r\n]>+/);
        my %parsed-utterances-by-speaker;
        my %speaks-about;
        for @lines -> $line {
            my ($speaker, $utterance) =
                parse-line-and-give-pair-speaker-utterance($line).kv;
            my ($line-logic, $speaks-about) = 
                parse-utterance-to-Logic-and-subjects($utterance).hash.{<logic speaks-about>};
            (%speaks-about{$speaker} //= []).push($speaks-about.list);
            (%parsed-utterances-by-speaker{$speaker} //= Logic.true)
                = %parsed-utterances-by-speaker{$speaker}.and($line-logic);
        }
        my @speaker = %speaks-about.keys;
        for %speaks-about.kv -> $speaker, $speaks-about {
            for $speaks-about.list.uniq -> $about {
                die "$speaker mentions $about but $about doesn't say anything."
                    unless @speaker.first({$_ eq $about});
            }
        }
        my @speaker-utterances =
            %parsed-utterances-by-speaker.kv.map({Logic.literal($^key).equiv($^value)});
        return @speaker-utterances.reduce({$^a.and($^b)});
    }

    sub parse-utterance-to-Logic-and-subjects($utterance) {
        my @speaks-about;
        my $logic;
        given $utterance {
            when / ^ (\w+) ' is a knight.' $ / {
                push @speaks-about, ~$0;
                $logic = Logic.literal(~$0);
            }
            when / ^ (\w+) ' is a knave.' $ / {
                push @speaks-about, ~$0;
                $logic = Logic.literal-negated(~$0);
            }
            when / ^ (\w+) ' and ' (\w+) ' are of the same type.' $ / {
                push @speaks-about, ~$0, ~$1;
                $logic = Logic.literal(~$0).equiv(Logic.literal(~$1));
            }
            when / ^ (\w+) ' and ' (\w+) ' are of different types.' $ / {
                push @speaks-about, ~$0, ~$1;
                $logic = Logic.literal(~$0).equiv(Logic.literal(~$1)).not;
            }
            default {
                die "Unrecognized utterance '$utterance'";
            }
        }
        return (:$logic, :speaks-about(@speaks-about));
    }

    sub parse-line-and-give-pair-speaker-utterance (Str $line) {
        my Str      $speaker;
        unless $line ~~ m/ ^ (\w+) ': ' / {
            die "Lines must be on the form '<name>: <utterance>'";
        }
        $speaker = ~$0;
        my $utterance = $line.substr(2 + $speaker.chars);
        return $speaker => $utterance;
    }

    multi sub say-solution-t1 (Logic $logic) {
        given $logic.conjuncts.elems {
            when 0 { say "No solutions."; }
            when 2..* { say "Many solutions:"; }
        }
        for $logic.conjuncts -> $conjunct {
            say $conjunct.literals.map({
                .name()  ~ ': ' ~ (.negated() ?? 'knave' !! 'knight')
            }).join('; ');
        }
    }
    multi sub say-solution-t1 (Any $) { }

    sub MAIN {
        my $logic = parse-stdin-to-Logic();
        say-solution-t1($logic);
        CATCH {
            default { .say; }
        }
    }

    # vim: nu ai et sts=4 sw=4 ts=4 ft=perl6

Logic.pm:

    use v6;

    class Literal {
        has Str  $.name = '';
        has Bool $.negated = False;

        method positive($name?) {
            return self.bless(*, :name($name // ''), :negated(False));
        }

        method negative($name?) {
            return self.bless(*, :name($name // ''), :negated(True));
        }

        method inverted() {
            self.bless(*, :name(self.name), :negated(! self.negated));
        }

        method isTrue() { return !self.name.chars && !self.negated; }
        method isFalse() { return !self.name.chars && self.negated; }

        method Str() { return '[L]' ~ self.name ~ '!' ~ !(self.negated // False); }
    }


    class Conjunct {
        has Literal @.literals;

        method new(Literal *@literals) {
            my $last = Literal.positive();
            my @tmp =
                gather for @literals.sort({ $^x.Str cmp $^y.Str }) -> $current {
                    if $current.isTrue {
                        # A /\ True == A
                        # ignore superfluous True
                        next;
                    }
                    elsif $current.isFalse {
                        # A /\ False == False
                        # signal False
                        take $current;
                        # stop further processing
                        last;
                    }
                    elsif $last.name ne $current.name {
                        # A /\ B
                        # concider both
                        $last = $current;
                        take $current;
                    }
                    elsif $last.negated != $current.negated {
                        # A /\ not(A) == False
                        take Literal.negative();
                        last;
                    }
                    # else: A /\ A == A; ignore duplicate
                };
            @tmp = @tmp[*-1] if @tmp.elems && @tmp[*-1].isFalse;
            return self.bless(*, literals => @tmp);
        }

        method and(Conjunct $c) {
            return Conjunct.new(self.literals, $c.literals);
        }

        method isTrue() {
            return self.literals.elems == self.literals.grep({.isTrue}).elems;
        }

        method isFalse() {
            return self.literals.grep({.isFalse}).elems > 0;
        }

        method Str() {
            return '[C]' ~ self.literals.map({.Str}).join(' /\ ');
        }
    }

    class Logic {
        has Conjunct @.conjuncts;

        method new(Conjunct *@conjuncts) {
            my %seen;
            return self.bless(*, :conjuncts(@conjuncts.grep({ ! %seen{~$_}++ })));
        }

        method true() {
            Logic.new(Conjunct.new(Literal.positive()));
        }

        multi method literal(Str $a) {
            Logic.new(Conjunct.new(Literal.positive($a)));
        }

        multi method literal(Literal $a) {
            Logic.new(Conjunct.new($a));
        }

        method literal-negated(Str $a) {
            Logic.new(Conjunct.new(Literal.negative($a)));
        }

        method not() {
            sub Conjunct-not(Conjunct $c) {
                my @l = $c.literals.map({.inverted}).map({Logic.literal($_).conjuncts});
                Logic.new(@l);
            }
            my @c = self.conjuncts.map({Conjunct-not($_)});
            @c.reduce({$^a.and($^b)});
        }

        multi method and(Logic $a: Logic $b = Logic.true) {
            return self.bless(*, conjuncts =>
                gather {
                    my @conjuncts = $a.conjuncts;
                    @conjuncts = Conjunct.new(Literal.positive()) unless @conjuncts.elems;
                    my %seen;
                    for @conjuncts -> $left-conjunct {
                        for $b.conjuncts -> $right-conjunct {
                            my Conjunct $and-conjunct = $left-conjunct.and($right-conjunct);
                            take $and-conjunct
                                unless $and-conjunct.isTrue
                                || $and-conjunct.isFalse
                                || %seen{~$and-conjunct}++; 
                        }
                    }
                }
            );
        }
        multi method and(Any $: Logic $b = Logic.true) { $b }

        method equiv(Logic $a: Logic $b) {
            $a.and($b).or($a.not.and($b.not));
        }

        method or(Logic $a: Logic $b) {
            Logic.new($a.conjuncts, $b.conjuncts);
        }

        method Str() {
            return '[D]' ~ self.conjuncts.map({'(' ~ .Str ~ ')'}).join(' \/ ');
        }

    }


    # vim: nu ai et sts=4 sw=4 ts=4 ft=perl6

## Correctness

The program runs correctly.

## Consistency

The style is clear and compact, and reads well.

## Clarity of intent

The comments are helpful.

You just gotta love a solution that supplies you with a module called
`Logic.pm`. The subtext almost being "there wasn't enough logic in the world,
so I made some and put it in this module". (Maybe the module should have been
called `Proposition.pm`, or something.)

Oh, look, `Logic` actually makes a little DSL there. Though it seems to me a
missed opportunity to put the `.not` at the end like that...

Besides that, the program clearly reaches its intended crescendo in the
subroutine `say-solution-t1`, where is basically loops through the solutions
and outputs them.

## Algorithmic efficiency

I have a hard time evaluating the efficiency here, though I suspect it's
exponential.

## Idiomatic use of Perl 6

`.grep({ ! %seen{~$_}++ }` is, if I'm not mistaken, a re-implementation of
`.uniq`.

`:speaks-about(@speaks-about)` feels like it was just one step from being
written `:@speaks-about`. The named before it was abbreviated in this way.

## Brevity

This solution falls somewhere in the middle in terms of length.
