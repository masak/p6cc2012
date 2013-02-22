    #! /usr/bin/env perl6

    use v6;

    constant KNIGHTS = ' T';
    constant KNAVES = ' F';

    grammar Dialog {
        token TOP { ^ <statement>* $ }
        token statement { <declaration> | <comparison> | <malformed> }
        token declaration { <islander> ':' <.ws> <islander> " is a " <type> \. \n? }
        token comparison { <islander> ':' <.ws> <islander> " and " <islander> " are of " <relation> <.ws> types? \. \n? }
        token malformed { <islander> ':' <.ws> (\N+) \n? }
        token islander { (\w+) }
        token type { knight | knave }
        token relation { "the same" | "different" }
    }

    sub ASSOCIATE      ($a, $b) { @*actions.push: ['knight', $a, $b] }
    sub DISASSOCIATE   ($a, $b) { @*actions.push: ['knave',  $a, $b] }
    sub CLAIM_SAME ($s, $a, $b) { @*actions.push: ['the same',  $s, $a, $b] }
    sub CLAIM_DIFF ($s, $a, $b) { @*actions.push: ['different', $s, $a, $b] }

    class Affiliation {
        has $.leader;
        has Bool $.type is rw;
        has Set $.members is rw;
        has @.unapplied-relationships is rw;
        has Affiliation $.other-group is rw;

        method new ($name, Bool $type?) {
            my $members = $type.defined ?? set() !! set($name);
            self.bless(*, :leader($name), :$members, :$type)
        }

        method merge (Affiliation $other) {
            self.members = $.members (|) $other.members;
            if $.type.defined {
                eager do for $other.unapplied-relationships -> $r {
                    $.type == $r[0] ?? ASSOCIATE($r[1],$r[2]) !! DISASSOCIATE($r[1],$r[2]);
                }
            }
            else {
                @.unapplied-relationships.push($other.unapplied-relationships);
            }
        }

        method is-affiliated (Affiliation $other) {
            if +($.members (&) $other.members) {
                return True;
            }
            if $.other-group.defined && +($.other-group.members (&) $other.members) {
                return False;
            }
            return;
        }
    }

    class Islander {
        has Str $.name;
        has Bool $.has-spoken is rw = False;
        has KeySet $.mentioned-by is rw = KeySet.new;
    }

    class Island {
        has Islander %.inhabitant;
        has Affiliation %.group;
        has %.unverified-statements;
        has $.debug is rw;

        submethod BUILD (:$debug) {
            self.debug = $debug;
            self.group{KNIGHTS} = Affiliation.new(KNIGHTS, True);
            self.group{KNAVES}  = Affiliation.new(KNAVES, False);
            self.disassociate(self.group.values);
        }

        method TOP($/) {
            self.assert-everyone-has-spoken;
            self.assert-no-islanders-playing-both-sides;
            make self.possibilities;
        }

        method assert-everyone-has-spoken () {
            eager do for %.inhabitant.values -> $i {
                unless $i.has-spoken {
                    die "{ $i.mentioned-by.keys.join(',') } mentions { $i.name } but { $i.name } doesn't say anything.";
                }
            }
        }

        method assert-no-islanders-playing-both-sides () {
            eager do for %.group.values -> $v {
                if $v.other-group.defined {
                    if +($v.members (&) $v.other-group.members) > 0 {
                        die "No solutions.";
                    }
                }
            }
        }

        method clone-possibilities(@p) {
            gather for @p {
                my $h = Hash.new(.kv);
                take $h;
            }
        }

        method possibilities () {
            my @permutations = {};
            
            for self.unique-pairings X (True, False) -> $affiliation, $type {
                if !$affiliation.type.defined || $type == $affiliation.type {
                    my @reality = @permutations;
                    if @reality[0]{$affiliation.members.pick}.defined {
                        #I want to just use .clone, but it doesn't seem to work for hashes in the version of Rakudo I'm currently using (2012.12-43)
                        @reality = self.clone-possibilities(@permutations);
                        @permutations.push(@reality);
                    }

                    for @reality -> $r {
                         eager $affiliation.members.keys.grep({/<alpha>/}).map({ $r{$_} = $type ?? 'knight' !! 'knave' });
                         if $affiliation.other-group.defined {
                            eager $affiliation.other-group.members.keys.grep({/<alpha>/}).map({ $r{$_} = $type ?? 'knave' !! 'knight' });
                         }
                    }
                }
            }

            gather for @permutations -> $p {
                if +$p {
                    take $p.pairs.sort({.key}).map({ "{.key}: {.value}" }).join('; ');
                }
            }
        }

        method unique-pairings () {
            my $unique-groups = KeySet.new(%.group.values.map: { .members.keys });
            my @pairings;
            while +$unique-groups > 0 {
                my $group = $unique-groups.pick;
                my $affiliation = %.group{$group};
                @pairings.push($affiliation);
                eager $affiliation.members.map: { $unique-groups.delete_key($_) };
                if $affiliation.other-group.defined {
                    eager $affiliation.other-group.members.map: { $unique-groups.delete_key($_) }
                }
            }
            return @pairings;
        }

        method islander ($/) {
            if not %.inhabitant{~$[0]}.defined {
                %.inhabitant{~$[0]} = Islander.new(:name(~$[0]));
                %.group{~$[0]} = Affiliation.new(~$[0]);
            }
            make %.inhabitant{~$[0]};
        }

        method statement ($/) {
            my ($action, *@islanders) = ($<comparison> || $<declaration>).ast.flat;

            @islanders[0].has-spoken = True;
            eager @islanders[1..*].map: { $_.mentioned-by{@islanders[0].name}++ };

            self.do: $action, @islanders.map({ .name });
        }

        method comparison ($/) {
            make (~$<relation>, $<islander>.map: { .ast });
        }

        method declaration ($/) {
            make (~$<type>, $<islander>.map: { .ast });
        }

        method malformed ($/) {
            die "Unrecognized utterance '$[0]'";
        }

        method do ($command, @islanders) {
            my @*actions = [$command, @islanders];
            repeat {
                my ($action, *@groups) = @*actions.pop.flat;
                if $.debug {
                    say "$action, {@groups.join(', ')}";
                }
                @groups .= map: { %.group{$_} }
                given $action[0] {
                    when 'knight'    { self.associate: @groups }
                    when 'knave'     { self.disassociate: @groups }
                    when 'the same'  { self.relate: @groups, True }
                    when 'different' { self.relate: @groups, False }
                }
            } while +@*actions;
        }

        method associate(@i) {
            my ($a, $b) = @i.sort: { .leader }
            if $a === $b {
                return;
            }
            self.fact: [$a, $b], True;

            eager $b.members.keys.map: { %.group{$_} = $a }
            $a.merge($b);
            if $a.other-group.defined && $b.other-group.defined {
                eager $b.other-group.members.keys.map: { %.group{$_} = $a.other-group }
                $a.other-group.merge($b.other-group);
            }
            $a.other-group = $a.other-group || $b.other-group;
        }

        method disassociate(*@i) {
            self.fact: @i, False;

            for @i, @i.reverse -> $a, $b {
                if $a.other-group.defined {
                    self.associate([$a.other-group, $b]);
                }
                else {
                    $a.other-group = $b;
                }
            }
        }

        method relate ([$speaker, *@i], $same) {
            my ($a, $b) = @i.sort: { .leader }
            if $speaker.type.defined {
                $speaker.type == $same ?? ASSOCIATE($a.leader, $b.leader) !! DISASSOCIATE($a.leader, $b.leader);
                return;
            }

            #I would override ACCEPTS, but this can also return an undefined value and that didn't feel right
            my $are-equal = $a.is-affiliated($b);
            if $are-equal.defined {
                ASSOCIATE($are-equal == $same ?? KNIGHTS !! KNAVES, $speaker.leader);
                return;
            }

            my %comparison = %.unverified-statements{$a.leader}{$b.leader} || {};
            if %comparison{$same}.defined {
                ASSOCIATE($speaker.leader, %comparison{$same});
            }
            elsif %comparison{!$same}.defined {
                DISASSOCIATE($speaker.leader, %comparison{!$same});
            }

            %comparison{$same} = $speaker.leader;
            %.unverified-statements{$a.leader}{$b.leader} = %comparison;
            $speaker.unapplied-relationships.push([$same, $a.leader, $b.leader]);
        }

        method fact (@groups, $are-same) {
            my ($a, $b) = @groups.map({ .leader }).sort;

            if %.unverified-statements{$a}.defined {
                my %comparisons = %.unverified-statements{$a}.delete($b) || {};
                if !+%.unverified-statements{$a} {
                    %.unverified-statements.delete($a);
                }
                if %comparisons{$are-same}.defined {
                    ASSOCIATE(KNIGHTS, %comparisons{$are-same});
                }
                elsif %comparisons{!$are-same}.defined {
                    ASSOCIATE(KNAVES, %comparisons{!$are-same});
                }
            }

            if !$are-same {
                return;
            }

            if %.unverified-statements{$b}.defined {
                for %.unverified-statements.delete($b).kv -> $other, $v {
                    for $v.kv -> $same, $islander {
                        #No objects as keys yet
                        $same ~~ 'True' ?? CLAIM_SAME($islander, $a, $other) !! CLAIM_DIFF($islander, $a, $other);
                    }
                }
            }

            eager do for %.unverified-statements.kv -> $first, $value {
                if $value{$b}.defined {
                    for $value.delete($b).kv -> $same, $islander {
                        $same ~~ 'True' ?? CLAIM_SAME($islander, $first, $a) !! CLAIM_DIFF($islander, $first, $a);
                    }
                }
            }
        }
    }

    sub parse($dialog, $out, $debug) {
        my $p = Dialog.parse($dialog, :actions(Island.new(:$debug)));
        if $p {
            if +$p.ast > 1 {
                $out.say('Many solutions:');
            }
            eager $p.ast.sort({ $^b leg $^a }).map: { $out.say($_) };
        }
        else {
            $out.say("Lines must be on the form '<name>: <utterance>'");
        }
        CATCH {
            when X::AdHoc { $out.say($_.payload) }
        }
    }

    multi MAIN {
        parse $*IN.slurp, $*OUT, False;
    }

    multi MAIN ($file, Bool :$v?) {
        #This main is useful for debugging with Rakudo::Debugger because the debugger takes all stdin as its input
        parse slurp($file).split("===\n")[0], $*OUT, $v;
    }

    multi MAIN(Bool :$test!, Bool :$v?) {
        use TestUtil;
        use Test;
        use File::Find;
        for find(dir => 't').grep(/txt$/) {
            my ($in, $expected) = slurp($_).split("===\n");
            my $actual = io;
            parse($in, $actual, $v);
            is($actual, $expected, $_);
        }
        done();
    }

TestUtil.pm:

    module TestUtil;

    class IOStr {
        has Str $.buf is rw;

        method say($str) {
            $.buf ~= $str;
            $.buf ~= "\n";
        }

        method Str() {
            return $.buf;
        }

        method gist() {
            return $.buf;
        }
    }

    sub io (Str $buf='') is export {
        return IOStr.new(:$buf);
    }

## Correctness

As far as I can see, this program correctly solves the problem.

## Consistency

No particular comments about consistency. Looks good.

## Clarity of intent

I'm not sure I see the point of naming the action class `Island`.

A `parse` subroutine preferably shouldn't do output.

The comment about cloning not working is likely a misunderstanding about how
deep the rabbit hole goes. If you have an array of hashes, and clone the array,
you get a *cloned* array of the *original* hashes. In other words, shallow
cloning. If you expect more, then you are blissfully unaware of how problematic
deep cloning can be.

## Algorithmic efficiency

The method `possibilities` seems to construct a cross product of all possible
universes. So this algorithm is exponential.

## Idiomatic use of Perl 6

Nice touch dependency-injecting the output like that. But `$*OUT` is a
dynamical variable, so essentially passing it around as arguments/parameters
feels like overkill. Could've just assigned to it in the `:test` `MAIN`.

## Brevity

This program feels overly long. Indeed, it's the longest of the bunch, even
discounting the `TestUtil.pm` module.
