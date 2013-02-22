    use v6;
    grammar Knive {
        regex TOP  { ^ \n* [ <stmt> [ \n | $ ] ]+ $ }
        regex stmt {
            <said=.name>
            [ ': ' || <form> ]:
            [ <a=.name>
                [ ' is a '                     [ <nite> | <nave> ]
                | ' and ' <b=.name> ' are of ' [ <simi> | <diff> ]
                ]
            || <bull>
            ]
        }
        regex name { \w+                }
        regex nite { 'knight.'          }
        regex nave { 'knave.'           }
        regex simi { 'the same type.'   }
        regex diff { 'different types.' }
        regex bull { ' '? (.+?) $$ }
        regex form { <?> }
    }
    class X::Form is Exception {
        method message() { q<Lines must be on the form '<name>: <utterance>'>; }
    }
    class X::Bull is Exception {
        has Str $.what;
        method message() { "Unrecognized utterance '$.what'"; }
    }
    class X::Mute is Exception {
        has Pair $.who;
        method message() {
            "{$.who.value} mentions {$.who.key} but {$.who.key} doesn't say anything."
        }
    }
    class Knive::Action {
        has %speakers; # Smullyans who made a statement
        has %mentions; # Smullyans mentioned in a statement
        has %surmises; # guesses about who is Knight or Knave
        has @findings; # statements made by Smullyans
        has $solution;
        
        method TOP($/) {
            for %mentions.grep: {!%speakers{$_.key}} {
                die X::Mute.new(:who($_));
            }
            self.solve((%speakers,%mentions)>>.keys.uniq.sort);
            if !defined $solution { say 'No solutions.'; }
            elsif $solution ne 'OK' { say $solution; } # when only 1 result
        }
        method stmt($_: $/) { 
            push @findings,
                .makeFinding(~$<said>, ?($<nite> | $<simi>), ~$<a>, (~$<b> if ?$<b>));
        }
        method bull($/) { die X::Bull.new(:what(~$0)); }
        method form($/) { die X::Form.new(); }
        
        method T($n) is rw { %surmises{$n}; } # get guess for given Smullyan
        method makeFinding($_: $said, $what, $a, $b) {
            %speakers{$said}++;
            %mentions{$a} //= $said;
            if defined $b {
                %mentions{$b} //= $said;
                -> { .T($said) == ($what == (.T($a) == .T($b))); }
            } else {
                -> { .T($said) == ($what ==  .T($a) ); }
            }
        }
        method solve($s: @smuls, $level = @smuls.elems - 1) {
            if $level < 0 {
                $s.solution: join '; ', @smuls.map({"$_: "~<knave knight>[self.T($_)]})
                    if all @findings>>.();
            } else {
                $s.T(@smuls[$level]) = True;
                $s.solve(@smuls, $level-1);
                $s.T(@smuls[$level]) = False;
                $s.solve(@smuls, $level-1);
            }
        }
        method solution($sol) {
            if !defined $solution { # store 1st solution
                $solution = $sol;
            } elsif $solution eq 'OK' {
                say $sol;
            } else { # multiple solutions, first print stored 1st solution
                say 'Many solutions:';
                say $solution;
                say $sol;
                $solution = 'OK';
            }
        }
    }
    {
    Knive.parse(slurp, :actions(Knive::Action.new));
    CATCH {when (X::Mute | X::Form | X::Bull) { say .message }}
    }

## Correctness

This program correctly solves the problem.

## Consistency

Overall a good impression. Dunno if I like the non-indentation of the two lines
of "main program"... but as quirks go, I'm prepared to accept it. Though to me
it'd actually been clearer without those braces around it.

## Clarity of intent

Most of the time very clear, and a bit funny.

The names `a` and `b` in the `stmt` rule are far too important to have been
called that. `speaker` and `speakee` would've worked, for example. Or `who`
and `whom`.

## Algorithmic efficiency

As evidenced by the `solve` method, this program tries all combinations of
true and false, in succession. So the algorithm is exponential.

## Idiomatic use of Perl 6

The names of rules and grammars alone somehow feel very Perl6ish. `:-)`

Also, it's a little cool that the *whole* task is solved through a grammar and its action class.

Nice use of invocation topicalization (`$_:`) in `makeFinding`. The same method
also produces callables, which are then hyper-called in `solve`. Much
appreciated.

No, wait, `if all @findings>>.()` is *brilliant*. I mean, look at it. The whole
program is really building up to that moment. And it is clearly a Perl 6 moment.

## Brevity

This program is not super-compact, but doesn't waste lines. There are no empty
lines, for example. The whole thing has the feel of a cohesive story.
