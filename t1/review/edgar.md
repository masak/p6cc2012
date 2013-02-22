    use v6;


    #########
    # Input #
    #########

    my @statements;

    sub read-input(Str $input) {

      grammar Knights::Grammar {
        regex TOP { ^ <line>* %% \n $ }

        regex line { <good-line> || <bad-line> }
        regex good-line { <ident> \s* ":" \s* <utterance> }
        regex bad-line { \N+ }

        regex utterance { <good-utterance> || <bad-utterance> }

        proto regex good-utterance { * }
        multi regex good-utterance:sym<knight> {
          <ident> " is a knight."
        }
        multi regex good-utterance:sym<knave> {
          <ident> " is a knave."
        }
        multi regex good-utterance:sym<same> {
          <ident> " and " <ident> " are of the same type."
        }
        multi regex good-utterance:sym<different> {
          <ident> " and " <ident> " are of different types."
        }

        regex bad-utterance { \N+ }
      }

      class Knights::Actions {
        method TOP($/) { make $<line>».ast; }

        method line($/) { make $<good-line>.ast }
        method good-line($/) { make $<utterance>.ast.(~$<ident>) }
        method bad-line($/) {
          die "Lines must be on the form '<name>: <utterance>'"
        }

        method utterance($/) { make $<good-utterance>.ast }

        method good-utterance:sym<knight>($/) {
          make { [ 'knight', $^who, ~$<ident> ] }
        }
        method good-utterance:sym<knave>($/) {
          make { [ 'knave', $^who, ~$<ident> ] }
        }
        method good-utterance:sym<same>($/) {
          make { [ 'same', $^who, ~$<ident>[0], ~$<ident>[1] ] }
        }
        method good-utterance:sym<different>($/) {
          make { [ 'different', $^who, ~$<ident>[0], ~$<ident>[1] ] }
        }

        method bad-utterance($/) { die "Unrecognized utterance '$/'" }
      }


      my $contents = ($input.defined ?? $input.IO.lines.join("\n") !! $*IN.slurp);
      my $actions = Knights::Actions.new();
      my $match = Knights::Grammar.parse($contents, :actions($actions));
      @statements = @($match.ast);
    }

    sub verify-input() {
      my %speakers = @statements.map: *[1] => 1;
      for @statements -> $s {
        for $s[2 .. *] -> $subject {
          if $subject !~~ %speakers {
            die "$s[1] mentions $subject but $subject doesn't say anything."
          }
        }
      }
    }


    #############
    # Reduction #
    #############

    class NoSolution is Exception { }

    # Dictionary converting names to indexes (0-based) and vice-versa.
    my %name-index;
    my @name-index;

    sub index-name(Str $name) {
      if $name !~~ %name-index {
        %name-index{$name} = +@name-index;
        @name-index.push($name);
      }
      return %name-index{$name}
    }

    # The problem is an instance of XOR-SAT, i.e., satisfactibility of
    # boolean formulas consisting in a conjunction of XOR'ed literals.
    # This problem can be reduced to solving a linear equation system in
    # Z/2Z (the finite field of size 2).
    #
    # The next two variables contain the coefficient matrix and the
    # independent term vector, so the problem will be solving:
    #
    #   @Amatrix * X = @Bvector
    #
    # Where the components of X with value 1 will correspond to knights,
    # and those with value 0, to knaves.
    my @Amatrix;
    my @Bvector;

    # Arithmetic in Z/2Z
    subset Z2Z of Int where { $^n == any(0, 1) }
    multi sub prefix:<(-)>(Z2Z $x) { 1 - $x }
    multi sub infix:<(+)=>(Z2Z $x is rw, Z2Z $y) { $x = ($x + $y) +& 1 }

    # Generate the linear system from the input list of statements.
    sub generate-system() {

      sub add-equation(@names, Z2Z $independent-term) {
        my @indices = @names.map: { %name-index{$_} };
        my $row = [ 0 xx +@name-index ];
        $row[$_] = (-) $row[$_] for @indices;

        if all($row[@indices]) == 0 {
          die NoSolution.new if $independent-term == 1;
        } else {
          @Amatrix.push($row);
          @Bvector.push($independent-term);
        }
      }

      # W: A is knight.
      #   W <-> A  =>  W xor A = 0
      multi sub generate-equation(['knight', $w, $a]) {
        add-equation([$w, $a], 0);
      }

      # W: A is knave.
      #   W <-> ~A  =>  W xor A = 1
      multi sub generate-equation(['knave', $w, $a]) {
        add-equation([$w, $a], 1);
      }

      # W: A and B are of the same type.
      #   W <-> (A <-> B)  =>  W xor A xor B = 1
      multi sub generate-equation(['same', $w, $a, $b]) {
        add-equation([$w, $a, $b], 1);
      }

      # W: A and B are of different types.
      #   W <-> (A <-> B)  =>  W xor A xor B = 0
      multi sub generate-equation(['different', $w, $a, $b]) {
        add-equation([$w, $a, $b], 0);
      }


      # We first index all the names, to know the total number of
      # variables, and then generate the equations.
      for @statements -> $s { index-name($_) for $s[1 .. *] }
      for @statements -> $s { generate-equation($s) }
    }


    ############
    # Solution #
    ############

    # Reduce the linear system using Gauss' method.
    sub reduce-system() {
      my $n-columns = +@name-index;
      my $r = 0;
      my $c = 0;
      while ($r < @Amatrix && $c < $n-columns) {
        # Find the first non-null element in this column, and in the rows
        # $r and below, to be used as pivot.
        my $pivot;
        for ($r ..^ @Amatrix) -> $p {
          if @Amatrix[$p][$c] {
            $pivot = $p;
            last;
          }
        }

        if $pivot.defined {
          # Swap rows if the pivot is not in row $r.
          if $pivot != $r {
            @Amatrix[$pivot, $r] = @Amatrix[$r, $pivot];
            @Bvector[$pivot, $r] = @Bvector[$r, $pivot];
          }

          # Reduce.
          for ($r ^..^ @Amatrix) -> $rr {
            if @Amatrix[$rr][$c] == 1 {
              for ($c ..^ $n-columns) -> $cc {
                @Amatrix[$rr][$cc] (+)= @Amatrix[$r][$cc];
              }
              @Bvector[$rr] (+)= @Bvector[$r];
            }
          }

          ++$r;
        }

        ++$c;
      }

      # All the remaining rows of the @Amatrix are zero. If any
      # corresponding independent term in @Bvector is not null, then we
      # have obtained a 0 = 1 equation, which allows no solutions.
      for ($r ..^ @Amatrix) -> $rr {
        die NoSolution.new if @Bvector[$rr] != 0;
      }

      # We can now remove the trailing null rows.
      @Amatrix.splice($r);
      @Bvector.splice($r);
    }

    my @solutions;

    # Construct the set of solutions.
    # At this point, @AMatrix will have a shape like:
    #   1 x x ...
    #   0 1 x ...
    #         ...
    #   0 0 0 ... 1 x x ...
    #   0 0 0 ... 0 0 1 ...
    #                   ...
    #   0 0 0 ... 0 0 0 ... 0 1 x
    sub construct-solutions() {
      my @current = [ 0 xx @name-index ];

      sub construct-solutions-rec(Int $r is copy, Int $c is copy) {
        loop {
          if $c == -1 {
            # We reached the leftmost column, and can store the solution.
            @solutions.push([ @name-index Z=> @current ]);
            last;

          } elsif $r == -1 || @Amatrix[$r][$c] == 0 ||
                  any(@Amatrix[$r][0 ..^ $c]) == 1 {
            # The current variable is unconstrained, so our solution
            # branches here, and we need to make two recursive calls.
            for 1, 0 -> $v {
              @current[$c] = $v;
              construct-solutions-rec($r, $c - 1);
            }
            last;

          } else {
            # The value of the current variable is determined by the
            # previous ones.
            my $value = @Bvector[$r];
            for $c + 1 ..^ @current -> $cc {
              if @Amatrix[$r][$cc] {
                $value (+)= @current[$cc];
              }
            }
            @current[$c] = $value;
            --$r; --$c;
          }
        }
      }

      construct-solutions-rec(@Amatrix - 1, @name-index - 1);
    }


    ##########
    # Driver #
    ##########

    sub display-solutions() {
      say "Many solutions:" if @solutions > 1;
      for @solutions -> $s {
        say ($s.map: { "{.key}: {.value ?? 'knight' !! 'knave'}" }).join('; ');
      }
    }

    multi sub MAIN(Str :$input) {
      read-input($input);
      if (@statements) {
        verify-input();
        generate-system();
        reduce-system();
        construct-solutions();
        display-solutions();
      }
      return 0;

      CATCH {
        when NoSolution {
          say 'No solutions.'
        }
        default {
          say ~$_;
        }
        return -1;
      }
    }

    # Local variables:
    # mode: cperl6
    # coding: utf-8
    # indent-tabs-mode: nil
    # End:

## Correctness

This program appears correct.

## Consistency

No particular comments here; looks good.

## Clarity of intent

The program is clearly sectioned into four parts.

It re-uses common terminology from linear algebra, which helps unload quite a
bit of mental burden from the reader.

## Algorithmic efficiency

This algorithm is polynomial, as opposed to all the other ones. It analyzes the
problem in matrix form, and only branches universes at the very last moment,
when generating solutions.

## Idiomatic use of Perl 6

Nice use of protoregexes and multis throughout. Also, a nice use of a recursive
sub in a sub.

Declaring your own arithmetic type, and operators over that type is... awesome,
and very p6ish.

## Brevity

This is one of the longer ones. It trades brevity for performance, though.
