    #!/usr/bin/env perl6
    use v6;
    use fatal;
    
    class Word is Cool {
        has Str $.str handles <elems Str gist>;
        has Int $.syllables;
    }
    
    my %move-to-front = <
    a           1
    an          1
    many        2
    one         1
    two         1
    three       1
    four        1
    five        1
    several     3
    >;
    
    # word lists inspired/partially taken from http://www.englishclub.com/vocabulary/regular-verbs-list.htm
    # and http://www.momswhothink.com/reading/list-of-nouns.html
    my %words = <
    abberrant   3     amusing     3    cable       2    candy       2
    ablaze      2     bad         1    cackle      2    canon       2
    abnormal    3     bare        1    cap         1    canyon      2
    ahead       2     bald        1    cat         1    capitalist  4
    achy        2     barbed      2    car         1    carbon      2
    altered     2     cabin       2    calif       2    carriage    3
    
    carrot      2     dalmatian   4    escaped     2    enforced    2
    cash        1     dancer      2    elaborated  4    engulfed    2
    causality   3     dandy       2    elapsed     2    escapes     2
    celebrity   3     dentist     2    elected     3    elects      2
    cell        1     daughter    2    emoted      3    elapses     3
    dachshund   2     daughters   2    endorsed    2    endorses    3
    enforces    3     engulfes    3
    
    face        1     fetch       1    force       1    galley      2
    fade        1     file        1    form        1    game        1
    fail        1     fill        1    found       1    gate        1
    fancy       2     film        1    frame       1    geese       1
    fasten      2     fire        2    frighten    2    ghost       1
    fax         1     fit         1    fry         1    giraffe     2
    fear        1     fix         1    glue        1    girl        1
    fence       1     flap        1    goldfish    2    glove       1
    
    goose       1     grain       1    grape       1    guitar      2
    governor    3     grandfather 3    grass       1    gun         1
    grade       1     grandmother 3    guide       1
    >;
    
    my @words =  (%words.kv, (%move-to-front.kv xx 2)).map: -> $w, $s {
        Word.new(str => $w, syllables => $s.Int);
    };
    
    my %syl;
    
    my %syl_req = (
        5 => 2,
        7 => 1,
    );
    
    sub score($_) {
        return 0 when %move-to-front;
        return 1 when m:i/^<[abw]>/;   # adjectives, adverbs
        return 2 when m:i/^<[efg]>/;   # verbs, particips
        return 3;
    }
    sub reorder(@words) {
        @words.sort: &score;
    }
    
    my @current;
    loop {
        given [+] @current>>.syllables {
            when * < 5  { @current.push: @words.roll }
            when 5|7    {
                my $chars = @current.chars;
                %syl{$_}{$chars}.push: [@current];
                @current = ();
                my $has-haiku = True;
                for %syl_req.kv -> $syl, $count {
                    unless %syl{$syl}{$chars}
                            && %syl{$syl}{$chars} >= $count {
                        $has-haiku = False;
                        last;
                    }
                }
                if $has-haiku {
                    say reorder %syl{5}{$chars}[0];
                    say reorder %syl{7}{$chars}[0];
                    say reorder %syl{5}{$chars}[1];
                    exit 0;
                }
            }
            default {
                @current.shift;
            }
        }
    }

## Correctness

A clear, straightforward approach. The central loop is an assembly line that
generates and consumes words. This appraoch may cause words to be repeated a
bit more than a fully random approach... but that's not necessarily a bad thing
because the brain likes repeated words.

It's also fairly easy to see that this algorithm will finish, by some
pidgeonhole argument or other. There's only so many lengths a five- or
seven-syllable line can have.

The program avoids the whole syllable-counting business by simply hardcoding the
syllable counts.

## Consistency

Nothing to really comment on here. A slight confusion whether to use hyphens or
underscores in variable names... but no biggie.

## Clarity of intent

I'm not entirely sure why `$.str` ended up handling `elems`.

## Algorithmic efficiency

One of the fastest solutions. Usually runs in between two and tree seconds on
my laptop.

## Idiomatic use of Perl 6

This is the kind of object orientation I absolutely dig. No big fanfare, no
bloated classes; just lift what would otherwise have been a more anemic data
structure into a class, call it `Word`, and it's all `Cool`. :)

Initializing hashes in that way is very Perl-y in general, I guess, but it
still feels idiomatic here.

The `&score` sub feels very sixish. Would've been tempting to move it
into the `reorder` sub, even.

## Brevity

The script is not the shortest of the bunch, but considering that it contains
its own word list as well, it's very self-contained.
