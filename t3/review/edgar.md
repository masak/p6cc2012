    use v6;


    #########
    # Input #
    #########

    my @target;

    sub parse-problem($arg) {
      @target = $arg.split('')».Int;
      die "Bad input" if not all(@target.sort «==» ^@target);
    }


    ############
    # Solution #
    ############

    enum Action <SAME UP DOWN>;

    my @solution;

    # Solve the problem.
    #
    # The sequence of actions is found using a parallel version of bubble
    # sort, in which multiple pairs of adjacent wires are swapped if their
    # positions with respect to the target sorting are reversed.
    #
    # For simplicity of the code (in particular, the comparisons), the
    # algorithm actually finds the sequence of actions needed to go from
    # @target to the sequential sorting 1 .. n, and then reverses them.
    sub solve-problem() {

      my @current;

      # Swap the pair of wires at positions $i, $i + 1.
      # The swap is performed in the earliest column possible.
      sub swap(Int $i) {
        @current[$i, $i + 1] = @current[$i + 1, $i];

        my $l = +@solution;
        --$l while $l > 0 && all(@solution[$l - 1][$i, $i + 1]) == Action::SAME;

        @solution.push: [ Action::SAME xx @current ] if $l == @solution;
        @solution[$l][$i, $i + 1] = Action::DOWN, Action::UP;
      }


      @current = @target;  # Start with the @target sorting.

      my $even = True;
      my $swapped-some = True;
      while $swapped-some {
        $swapped-some = False;
        if $even {
          for 0, * + 2 ...^ * >= @current.end -> $i {
            if @current[$i] > @current[$i + 1] {
              swap($i);
              $swapped-some = True;
            }
          }
        } else {
          for 1, * + 2 ...^ * >= @current.end -> $i {
            if @current[$i] > @current[$i + 1] {
              swap($i);
              $swapped-some = True;
            }
          }
        }
        $even = !$even;
      }

      @solution = @solution.reverse;
    }


    ##########
    # Driver #
    ##########

    sub display-solution() {
      for @target.kv -> $src, $tgt {
        print "$src _";
        for @solution -> $step {
          given $step[$src] {
            when Action::SAME { print '__' }
            when Action::UP   { print '/\\'  }
            when Action::DOWN { print '  ' }
          }
        }
        print "_ $tgt\n";

        if $src < @target - 1 {
          my $line = '   ';
          for @solution -> $step {
            given $step[$src] {
              when Action::SAME | Action::UP { $line ~= '  ' }
              when Action::DOWN { $line ~= '\\/' }
            }
          }
          $line ~~ s/\s+$//;
          print "$line\n";
        }
      }
    }

    multi sub MAIN($arg) {
      parse-problem($arg);
      solve-problem();
      display-solution();
      return 0;

      CATCH {
        say ~$_;
        return -1;
      }
    }

    # Local variables:
    # mode: cperl6
    # coding: utf-8
    # indent-tabs-mode: nil
    # End:

## Correctness

It's correct.

This solution prefers to do flips as far right as possible.

I notice that the solution is quite slow. I don't know why.

It also doesn't score fully on elegance. For example, for the input
`0816245397` it produces an eight-column solution where a seven-column solution
is possible:

    $ perl6 edgar 0816245397
    0 __________________ 0

    1 _______________  _ 8
                     \/
    2 _____________  /\_ 1
                   \/
    3 _________  __/\  _ 6
               \/    \/
    4 _______  /\  __/\_ 2
             \/  \/
    5 _____  /\  /\  ___ 4
           \/  \/  \/
    6 ___  /\  /\__/\  _ 5
         \/  \/      \/
    7 _  /\__/\______/\_ 3
       \/
    8 _/\____________  _ 9
                     \/
    9 _______________/\_ 7

As the `bruce`, `moritz` and `pkai` programs show, the 8 wire can do all flips
instead of stopping once like it does here.

## Consistency

No complaints here.

## Clarity of intent

Here's why I even *have* a "clarity of intent" section:

    parse-problem($arg);
    solve-problem();
    display-solution();
    return 0;

Very nice &mdash; looks kind of like a TODO list. I belong to the school of
thought that says if your code doesn't look like this, *make* it look like
this.

On the other hand, I don't think I'll ever understand what a comment such as

    ##########
    # Driver #
    ##########

actually adds to the program. I'll just write it off as "people code
differently, and some just like to put words in a box".

## Idiomatic use of Perl 6

These lines:

    my $swapped-some = True;
    while $swapped-some {
      $swapped-some = False;

Could've been better written as a `repeat while`:

    repeat while my $swapped-some {
        $swapped-some = False;

Skipping the need for the initial assignment just to get the loop started.

## Brevity

It's worth noticing that this script is 124 lines with commens, and 94 without.
