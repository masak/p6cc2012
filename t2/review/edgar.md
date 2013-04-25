    use v6;
    
    
    ###########
    # Grammar #
    ###########
    
    # Verse generation grammar.
    #
    # The terminal POS tagset is that from standard Penn treebank one,
    # with the additions of the (rather ad-hoc) tags:
    #   * DTS: Plural determiner.
    #   * MDZ: Modal verb in singular third-person form.
    #   * RBN: Negation adverb.
    #
    # The non-terminals are:
    #   * NP: Noun phrase.
    #   * NP1: Singular noun phrase.
    #   * NPS: Plural noun phrase.
    #   * NP1noPP: Singular noun phrase with no embedded PP.
    #   * NPSnoPP: Plural noun phrase with no embedded PP.
    #   * PP: Prepositional phrase.
    #   * AdjP: Adjective phrase.
    #   * AdvP: Adverb phrase.
    #   * VP1: Singular verb phrase.
    #   * VPS: Plural verb phrase.
    #   * MainVB1: Singular main verb group.
    #   * MainVBS: Plural main verb group.
    my %grammar =
      Verse   => [ [<$NP1>],
                   [<$NP1? $VP1>],
                   [<PRP $VP1>],
                   [<$NPS? $VPS>],
                   [<PRPS $VPS>],
                   [<$AdjP>],
                   [<$AdvP>] ],
      NP      => [ [<$NP1>],
                   [<$NPS>] ],
      NP1     => [ [<DT? $AdjP? NN $PP?>] ],
      NPS     => [ [<DTS? $AdjP? NNS $PP?>] ],
      NP1noPP => [ [<DT? $AdjP? NN>] ],
      NPSnoPP => [ [<DTS? $AdjP? NNS>] ],
      PP      => [ [<IN $NP>] ],
      AdjP    => [ [<RB? JJ>] ],
      AdvP    => [ [<RB? RB>] ],
      VP1     => [ [<$MainVB1 $AdvP? $NP? $PP?>] ],
      VPS     => [ [<$MainVBS $AdvP? $NP? $PP?>] ],
      MainVB1 => [ [<VBZ>],
                   [<MDZ RBN? VB>] ],
      MainVBS => [ [<VB>],
                   [<MD RBN? VB>] ];
    
    # Pick a derivation at random from the grammar.
    sub pick-from-grammar(Str $symbol is copy) {
      if $symbol ~~ /^(.+)\?$/ {
        # Optional production.
        $symbol = ~$0;
        if (True, False).pick {
          return pick-from-grammar($symbol);
        } else {
          return [];
        }
    
      } elsif $symbol ~~ /^\$(.+)$/ {
        # Non-terminal.
        $symbol = ~$0;
        die "Unexisting non-terminal $symbol" unless %grammar{$symbol} :exists;
    
        my $rhs = %grammar{$symbol}.pick;
        my $expansion = [];
        for @($rhs) -> $rhs-symbol {
          $expansion.push: @(pick-from-grammar($rhs-symbol));
        }
        return $expansion;
    
      } else {
        # Terminal.
        return [$symbol];
      }
    }
    
    
    #####################
    # Syllable counting #
    #####################
    
    # Adapted from CPAN Lingua::EN::Syllable-0.251
    
    # Patterns which substract one syllable from the count.
    constant @SUB_SYLLABLE =
      rx/cial/, rx/tia/, rx/cius/, rx/cious/, rx/giu/, rx/ion/, rx/iou/, rx/sia$/,
      rx/[<-[aeiou]>|<[gq]>u]ely$/;
    
    # Patterns which add one syllable to the count.
    constant @ADD_SYLLABLE =
      rx/ia/, rx/riet/, rx/dien/, rx/iu/, rx/io/, rx/ii/, rx/<[aeiouym]>bl$/,
      rx/<[aeiou]> ** 3/, rx/^mc/, rx/ism$/, rx/(<[^aeiouy]>)$0l$/,
      rx/<[^l]>lien/, rx/^coa<[dglx]>./, rx/<[^gq]>ua<[^auieo]>/, rx/dnt$/;
    
    # Number of syllables in a word.
    sub syllable-count(Str $word is copy) {
      return 1 if $word.codes < 3;  # 1- and 2-letter words have one syllable.
    
      $word .= lc;
      $word ~~ s:g/\'//;
      $word ~~ s/e$//;
    
      # Find number of vowel groups, and correct using the pattern lists.
      my $syllables = +($word.comb: /<[aeiouy]>+/);
      for @SUB_SYLLABLE -> $re { --$syllables if $word ~~ $re }
      for @ADD_SYLLABLE -> $re { ++$syllables if $word ~~ $re }
      $syllables max= 1;  # At least one syllable
    
      return $syllables;
    }
    
    
    ###########
    # Lexicon #
    ###########
    
    # Lexicon, indexed by POS and number of syllables.
    my %lexicon;
    
    # Read the lexicon, performing expansions such as plural, adverb or
    # singular third-person form generation.
    sub read-lexicon(Str $file) {
    
      sub add-word(Str $pos, Str $word) {
        my $syllables = syllable-count($word);
        %lexicon{$pos}{$syllables}.push: $word;
      }
    
      sub add-derived(Str $pos, @fields, Str $suffix = '') {
        if @fields > 2 {
          add-word($pos, @fields[2]) unless @fields[2] eq '-';
        } else {
          add-word($pos, @fields[1] ~ $suffix);
        }
      }
    
      for $file.IO.lines {
        my $line = $_;
        $line ~~ s/\#.+$//;
        my @fields = $line.comb: /\S+/;
        if @fields {
          add-word(@fields[0], @fields[1]);
          given @fields[0] {
            when 'DT' { add-derived('DTS', @fields) }
            when 'JJ' { add-derived('RB',  @fields, 'ly') }
            when 'MD' { add-derived('MDZ', @fields) }
            when 'NN' { add-derived('NNS', @fields, 's') }
            when 'VB' { add-derived('VBZ', @fields, 's') }
          }
        }
      }
    }
    
    # Pick a word at random.
    sub pick-word(Str $pos, Int $syllables) {
      die "Unexisting POS $pos" if not %lexicon{$pos} :exists;
      if %lexicon{$pos}{$syllables} :exists {
        return %lexicon{$pos}{$syllables}.pick;
      } else {
        return;
      }
    }
    
    
    ##############
    # Generation #
    ##############
    
    # Integer partitions of 5 and 7, indexed by their number of elements.
    # They have been precomputed for simplicity.
    my %partitions =
      5 => {
        1 => [ [5] ],
        2 => [ [4,1], [3,2] ],
        3 => [ [3,1,1], [2,2,1] ],
        4 => [ [2,1,1,1] ],
        5 => [ [1,1,1,1,1] ],
      },
      7 => {
        1 => [ [7] ],
        2 => [ [6,1], [5,2], [4,3] ],
        3 => [ [5,1,1], [4,2,1], [3,3,1], [3,2,2] ],
        4 => [ [4,1,1,1], [3,2,1,1], [2,2,2,1] ],
        5 => [ [3,1,1,1,1], [2,2,1,1,1] ],
        6 => [ [2,1,1,1,1,1] ],
        7 => [ [1,1,1,1,1,1,1] ],
      };
    
    # Parameters to determine the maximum number of tries for the various
    # parts of the generation process.
    constant MAX_TRIES_SYLL_SEQ = 5;
    constant MAX_TRIES_SYLL_SEQ_SHUFFLE = 5;
    
    # Generate a verse of the given number of syllables.
    sub generate-verse(Int $syllables where * == any(5, 7)) {
    
      # Fix minor spelling issues:
      # * Convert 'a' to 'an' if following word starts with a vocal.
      # * Join 'can' and 'not'.
      sub fix-spelling(@verse) {
        my $i = 0;
        while $i < @verse - 1 {
          if @verse[$i] eq 'a' and @verse[$i + 1] ~~ /^[aeiou]/ {
            @verse[$i] = 'an';
          }
          elsif @verse[$i] eq 'can' and @verse[$i + 1] eq 'not' {
            @verse[$i] = 'cannot';
            @verse.splice($i + 1, 1);
          }
          ++$i;
        }
      }
    
      # The verse generation process follows these steps:
      #   * A POS sequence is generated by picking from the $Verse grammar.
      #   * A partition of the desired number of syllables with as many
      #     elements as the number of tokens in the generated POS sequence
      #     is picked.
      #   * The elements in the partition are shuffled, and then paired with
      #     the corresponding tokens in the POS sequence.
      #   * Word with the given POS and number of syllables are picked.
      #   * If all the POS-syllables combinations produced one word, the
      #     resulting word sequence is checked for minor spelling fixes, and
      #     returned as resulting verse.
      #   * Otherwise, the partition shuffling is repeated up to
      #     MAX_TRIES_SYLL_SEQ_SHUFFLE times.
      #   * If the reshuffling did not produce a verse, the syllable
      #     sequence picking is repeated up to MAX_TRIES_SYLL_SEQ times.
      #   * If the repicking did not produce a verse, a new POS sequence is
      #     picked from the grammar, until a verse is produced.
      loop {
        my $pos-seq = pick-from-grammar('$Verse');
        redo unless 0 < $pos-seq <= $syllables;
    
        for ^MAX_TRIES_SYLL_SEQ {
          my $syll-seq = %partitions{$syllables}{+$pos-seq}.pick;
          for ^MAX_TRIES_SYLL_SEQ_SHUFFLE {
            my $syll-seq-order = $syll-seq.pick: *;
    
            my $success = True;
            my @verse;
            for $pos-seq Z $syll-seq -> $pos, $syll {
              my $word = pick-word($pos, $syll);
              if not $word.defined {
                $success = False;
                last;
              }
              @verse.push: $word;
            }
    
            if $success {
              fix-spelling(@verse);
              return @verse.join(' ');
            }
          }
        }
      }
    }
    
    # Generate a Haiku.
    #
    # Verses of 5 and 7 syllables are generated, and indexed by their
    # number of characters. Once two verses of 5 syllables and one of 7
    # have been produced for a given length, they are returned as Haiku.
    sub generate-haiku() {
      my %verses;
      loop {
        for 5, 7, 5 -> $syllables {
          my $verse = generate-verse($syllables);
          my $length = $verse.codes;
          %verses{$length}{$syllables}.push: $verse;
          if %verses{$length}{5} :exists and %verses{$length}{7} :exists and
             %verses{$length}{5} >= 2 and %verses{$length}{7} >= 1 {
            my @v5 = %verses{$length}{5}.pick: 2;
            my $v7 = %verses{$length}{7}.pick;
            return "@v5[0]\n$v7\n@v5[1]\n";
          }
        }
      }
    }
    
    
    ##########
    # Driver #
    ##########
    
    multi sub MAIN() {
      read-lexicon('lex.txt');
      print generate-haiku();
      return 0;
    }
    
    # Local variables:
    # mode: cperl6
    # coding: utf-8
    # indent-tabs-mode: nil
    # End:

## Correctness

Besides which, stylistically and qualitatively, none of the other solutions
even come near this one. This solution is the real deal.

The central algorithm is this, simplified:

    Generate a sequence of parts-of-speech
        Given the length of the sequence, find a partition with that length
            Repeatedly shuffle the partition and for each element
                Try to find a word of that part-of-speech and of that length

This is repeated until two five-syllable lines and one seven-syllable line,
all three of the same length, are found.

## Consistency

The program uses domain terms from *actual* grammar. ("POS" means "part of
speech", by the way.)

## Clarity of intent

I just want to say that it's wonderful in a haiku generator to find a data
structure that contains all the integer particions of 5 and 7. That feels like
a kind of poetry, too.

The `splice` on line 213 could have easily subsumed the assignment on the line
before it. (And it would have been just as clear or clearer.)

## Algorithmic efficiency

This script has the idea of storing up same-length lines until enough are
found. Fairly efficient.

## Idiomatic use of Perl 6

The `syllable-count` is a really nice, Perl 6-faithful port of the CPAN module.
Kudos.

Nested subroutines!

Author tends to lean towards storing arrays in scalar variables. For variables
like `$expansion` and `$pos-seq`, I'd have definitely gone with arrays:
`@expansion` and `@pos-seq`. Now, clearly that's a matter of taste. But both
the language itself and the eye helps you a little if you do.

## Brevity

The author has not elected to make brevity a priority. That said, the lines
do not feel wasted, either.

## External resources

The file `lex.txt`:

    # The lexicon here tries to capture the traditional Haiku themes of
    # nature and nostalgy.

    # Pronouns
    PRP he
    PRP she
    PRP it
    PRPS I  # As if it were plural
    PRPS they
    PRPS we
    PRPS you

    # Determiners
    DT a -
    DT the
    DT my
    DT your
    DT his
    DT her
    DT its
    DT our
    DT their

    # Nouns
    NN bird
    NN cat
    NN cloud
    NN cow
    NN darkness
    NN dog
    NN fish fish
    NN flower
    NN goose geese
    NN grass
    NN horse
    NN house
    NN moon
    NN mouse mice
    NN rain
    NN sheep sheep
    NN sky skies
    NN sleep
    NN slumber
    NN snow
    NN star
    NN storm
    NN sun
    NN tear
    NN wind

    # Adjectives
    JJ bad
    JJ big
    JJ bitter
    JJ black
    JJ blue
    JJ cold
    JJ early -
    JJ frozen
    JJ good well
    JJ green
    JJ grey
    JJ hard
    JJ late
    JJ maroon
    JJ modest
    JJ pink
    JJ red
    JJ sad
    JJ small smally
    JJ smooth
    JJ soft
    JJ strong
    JJ warm
    JJ white
    JJ yellow

    # Adverbs
    RB almost
    RB always
    RB so
    RB very

    # Negation adverb
    RBN never
    RBN not

    # Modal verbs
    MD can
    MD could
    MD do does
    MD should
    MD will
    MD would

    # Verbs
    VB dream
    VB drink
    VB eat
    VB fall
    VB forget
    VB freeze
    VB hide
    VB move
    VB remember
    VB rise
    VB run
    VB set
    VB shout
    VB sing
    VB sleep
    VB warm

    # Prepositions
    IN at
    IN in
    IN for
    IN of
    IN to
