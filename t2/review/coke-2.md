    #!/usr/bin/env perl6
    
    use lib 'lib';
    use Lingua::EN::Syllable;
    
    # Read in some words to play with:
    
    my $lexicon = "words";
    my $syllabry = "syllables";
    
    # precompile our syllable file
    my %syllables;
    
    # only count syllables once. it's expensive. save off the counts.
    if ! $syllabry.IO.e  {
        # Tell the user what we're doing - except the tests don't like
        # anything on stderr.
        # note "compiling $syllabry - this could take a while...";
    
        my @words = $lexicon.IO.lines();
    
        for @words -> $word {
            my $count = syllable($word);
            next if $count > 7; # don't bother, we can't ever use it.
            %syllables{$count}.push($word);
        }
        my $fh = open($syllabry, :w);
        $fh.print(%syllables.perl);
        $fh.close;
    } else {
        # TODO - the eval is slowish. We might do better if we emitted a
        # leaner format. Resea---out of time.
        %syllables = eval slurp $syllabry;
    }
    
    sub getLine($count) {
        my @keysizes = keys %syllables;
    
        my @words;
    
        my $limit = 2;
        if $count > $limit {
            @words = getLine($limit), getLine($count-$limit);
        } elsif grep {$_ == $count}, @keysizes {
            @words = %syllables{$count}.pick(1);
        } else {
            @words = getLine(1), getLine($count-1);
        }
        return join(" ",@words);
    }
    
    sub MAIN() {
        my %lines;
    
        loop {
            # Get two fives and a seven.
            for <5 7 5> -> $x {
                my $line = getLine($x);
                my $length = $line.chars; 
        
                # manually vivify.
                for <5 7> -> $y {
                    if !defined(%lines{$y}{$length}) {
                        %lines{$y}{$length} = Array.new();
                    }
                } 
                push(@(%lines{$x}{$length}), $line);
        
                my @fives  = %lines<5>{$length}.list;
                my @sevens = %lines<7>{$length}.list;
    
                # Did these line give us enough to make a haiku?
                if @fives.elems >= 2 && @sevens.elems {
                    @fives  = @fives.pick(*);
                    @sevens = @sevens.pick(*);
    
                    say @fives[0];
                    say @sevens[0];
                    say @fives[1];
                    exit 0;
                }
            }
        }
    }

Though this code shares a lineage with the author's previous submission, the
diff is large enough that the new code merits an entirely new review.

## Correctness

The program correctly generates rectangular haikus. It also uses the trick of
storing up old lines until it has enough.

## Consistency

Looks very pleasant and consistent to me.

## Clarity of intent

I like the comments on this one. They're informative.

## Algorithmic efficiency

Again, a speed gain by stratifying lines by syllables and then length. Just
like the previous version.

The thing runs in on the order of 20 seconds on my laptop.

I'm actually hard-pressed to propose a faster approach to loading something
than `eval slurp`. Would love hearing of one.

## Idiomatic use of Perl 6

`Array.new()` made me sit up a bit. What's wrong with `[]`?

## Brevity

Nice and short.

## External resources

Same as in the first submission. Only a slight difference in the diff:

    --- coke/lib/Lingua/EN/Syllable.pm  2013-04-07 21:08:51.918861941 +0200
    +++ coke-2/lib/Lingua/EN/Syllable.pm    2013-04-07 21:08:51.914861941 +0200
    @@ -1,5 +1,6 @@
    -#
     # This is a near-verbatim translation from the p5 module of the same name.
    -#
    +# Some words don't work at all, but we cleverly don't include them in
    +# our corpus. Cool, you know, unless masak is reading these comments.
    +################################################################################
     
     # basic algortithm:

The plan is foolproof, boss! masak is known to overlook things all the time!
