    use v6;


    #########
    # Input #
    #########

    my @cubes;

    sub read-input(Str $input) {

      grammar Rain::Grammar {
        regex TOP { ^ <line>* %% \n $ }
        regex line { <good-line> || <bad-line> }
        rule good-line { '(' <number> ** 3 % [ ',' ] ')' }
        regex bad-line { \N+ }
        token number { (<[+\-]>? \d+) }
        token ws { \h* }
      }

      clasg Rain::Actions {
        method TOP($/) { make $<line>».ast }
        method line($/) { make $<good-line>.ast }
        method good-line($/) { make $<number>».ast }
        method bad-line($/) { die "Unrecognized line: \"{~$/}\"" }
        method number($/) { make $0.Int }
      }


      my $contents = ($input.defined ?? $input.IO.lines.join("\n") !! $*IN.slurp);
      my $actions = Rain::Actions.new;
      my $match = Rain::Grammar.parse($contents, :actions($actions));
      @cubes = @($match.ast);
    }


    ############
    # Solution #
    ############

    # This solution is based in level rising of connected water masses.
    #
    # Iteratively, each mass of water is considered to check if it can
    # expand horizontally (possibly merging with adjacent water masses and
    # creating falls which may give birth to new water masses), and if its
    # level can rise. If no position in the surface of the water mass has
    # a fall as its neighbour, then we can rise the level one step, and
    # increase the volume.
    #
    # The surface of the water mass can change as result of horizontal
    # expansions, or when the surface is covered by cubes as the level
    # rises.
    #
    # The process is iterated while there are changes in the water mass
    # distribution.
    sub solve-problem() {

      # Water masses.
      my %water-masses;

      # Content of each position.
      my %contents;

      # Heights at which cubes are present in each column.
      my %columns;

      # (Dummy) representation of a cube.
      class Cube { }

      # Representation of a connected water mass.
      class WaterMass is rw {

        # Global ID for identification.
        my $next-id = 0;  # next one to be assigned
        has $.id;

        # Volume of water contained in this mass.
        has Int $.volume;

        # We only need to record the (x, z) coordinates of the positions
        # in the surface. The volume of water lying below does not take a
        # part in the rising process.
        has @.surface-xz;

        # y coordinate of the surface (level of the water).
        has Int $.level-y;

        method new(Int $x, Int $y, Int $z) {
          self.bless(*, id => $next-id++, volume => 0,
                     surface-xz => [ $x, $z ], level-y => $y);
        }

        # Index into the %water-masses and %contents hashes.
        method index() {
          %water-masses{$.id} = self;
          for @.surface-xz -> $x, $z {
            %contents{$($x, $.level-y, $z)} = self;
          }
        }

        # Merge with a neighbouring water mass.
        method !merge(WaterMass $other) {
          return if $other === self;
          die "Merging non-balanced water masses" if $other.level-y != $.level-y;

          for @($other.surface-xz) -> $x, $z {
            %contents{$($x, $.level-y, $z)} = self;
          }

          @.surface-xz.push: @($other.surface-xz);
          $.volume += $other.volume;
          %water-masses{$other.id} :delete;
        }

        # Expand the water mass to the given neighbouring position.
        method !expand-to(Int $n-x, Int $n-z) {
          my $changed = False;
          my $can-rise = False;

          my $yy = topmost-cube($n-x, $.level-y, $n-z);
          if $yy.defined {
            if $yy == $.level-y - 1 {
              # There is a cube just below, expand.
              @.surface-xz.push: $n-x, $n-z;
              %contents{$($n-x, $.level-y, $n-z)} = self;
              $can-rise = True;

            } elsif not %contents{$($n-x, $yy + 1, $n-z)} :exists {
              # Create a new water mass.
              my $mass = WaterMass.new($n-x, $yy + 1, $n-z);
              $mass.index;
              $changed = True;
            }
          }

          return $changed, $can-rise;
        }

        # Rise the water one level, with the given surface.
        method !rise(@risen-surface-xz) {
          $.volume += +@.surface-xz div 2;
          @.surface-xz = @risen-surface-xz;
          ++$.level-y;

          for @risen-surface-xz -> $x, $z {
            %contents{$($x, $.level-y, $z)} = self;
          }
        }

        # Single expansion and rise cycle.
        method expand-and-rise() {
          my $changed = False;
          my $can-rise = True;

          # Surface that would be exposed if we rose one level.
          my @risen-surface-xz;

          my $i = 0;
          while $i < @.surface-xz {
            my ($x, $z) = @.surface-xz[$i, $i + 1];

            # Check if we would hit a roof if we rose.
            my $up = ($x, $.level-y + 1, $z);
            if not %contents{$up} :exists {
              @risen-surface-xz.push: ($x, $z);
            }

            # Check our neigbouring positions.
            my @neighbours =
              ($x - 1, $z), ($x + 1, $z), ($x, $z - 1), ($x, $z + 1);
            for @neighbours -> $n-x, $n-z {
              my $n-pos = ($n-x, $.level-y, $n-z);
              if %contents{$n-pos} :exists {
                self!merge(%contents{$n-pos}) unless %contents{$n-pos} ~~ Cube;
              } else {
                my ($ch, $cr) = self!expand-to($n-x, $n-z);
                $changed   ||= $ch;
                $can-rise &&= $cr;
              }
            }

            $i += 2;
          }

          if $can-rise {
            self!rise(@risen-surface-xz);
            $changed = True;
          }

          return $changed;
        }
      }

      # Index the starting cubes into %contents and %columns.
      sub starting-contents() {
        for @cubes -> $xyz {
          my ($x, $y, $z) = @($xyz);
          my $xz = ($x, $z);
          %contents{$xyz} = Cube.new;
          %columns{$xz}.push: $y;
        }

        for %columns.keys -> $xz {
          %columns{$xz} .= sort;
        }
      }

      # Create the starting water masses at the positions immediately
      # above the highest cube in each column (which are the positions
      # which will directly receive the rain).
      sub starting-masses() {
        for %columns.kv -> $xz, @column {
          my ($x, $z) = $xz.split(' ')».Int;
          WaterMass.new($x, @column[* - 1] + 1, $z).index;
        }
      }

      # Find the top-most cube in a given column, at height lower or equal
      # than $y. Returns an undefined value if no such cube exists.
      sub topmost-cube(Int $x, Int $y, Int $z) {

        # Lower bound by binary search.
        sub lower-bound(@column, Int $y) {
          my $lo = 0;
          my $hi = @column;
          while $lo < $hi - 1 {
            my $mi = ($hi + $lo) div 2;
            if @column[$mi] > $y {
              $hi = $mi;
            } else {
              $lo = $mi;
            }
          }

          if $lo == 0 && $y < @column[0] {
            return;
          } else {
            return @column[$lo];
          }
        }

        my $xz = ($x, $z);
        if %columns{$xz} :exists {
          my $column = %columns{$xz};
          return lower-bound($column, $y);
        } else {
          return;
        }
      }


      starting-contents();
      starting-masses();

      my $some-changed = True;
      while $some-changed {
        $some-changed = False;
        my @current-water-masses = %water-masses.values;
        for @current-water-masses -> $mass {
          if %water-masses{$mass.id} :exists {  # It might have been merged.
            $some-changed = True if $mass.expand-and-rise;
          }
        }
      }

      return [+] %water-masses.values».volume;
    }


    ##########
    # Driver #
    ##########

    sub MAIN(Str :$input) {
      read-input($input);
      say solve-problem();
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

This program is almost but not entirely correct. It gives an error for the
following container, which should be OK.

    XXXX~XXXX
    X~~~~~~~X
    X~~XXX~~X
    X~~~~~~~X
    XXXXXXXXX

The error `Merging unbalanced water masses` is from line 104, and I suspect
it's because the water is computed as rising first on one side, and then it
"flows over" to the other side in a non-symmetric way. Would love to get
confirmation on this.

Having said that, this program is clearly the *least* incorrect, and just as
clearly the one where most energy has been spent in trying to make it correct.

## Consistency

I found no particular inconsistencies.

## Clarity of intent

I'm not sure I need comments like these in the program.

      # Water masses.
      my %water-masses;

It's nice that attributes are documented, and it's nice that the attribute has
a descriptive name... but I don't need the comment if it says exactly the same
as the attribute name.

## Idiomatic use of Perl 6

Nice, a grammar and an actions class inside of a sub &mdash; because that's the
only place where they're used. To finish off the pattern, should've declared
them both lexical with `my`, though.

While we're in that sub, `:actions($actions)` can be spelled shorter as
`:$actions`. That's a straightforward refactor waiting to happen &mdash; the
variable is correctly named for it and everything.

## Brevity

Given what it does, the program is actually surprisingly short.
