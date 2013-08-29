    use v6;

    my $input = [ @*ARGS.split('') ];
    my @toPrint = (('_ '~$_,'   ') for @$input);
    loop {	
        my ($tmpArr,$change) = sort_1_step($input);
        if $change.elems == 0 {
            @toPrint[2*$_,2*$_+1] = ($tmpArr[$_]~' _','   ') Z~ @toPrint[2*$_,2*$_+1] for 0..$input.end;
            last;
        } else {
            my $n = 0;
            while $n <= $input.end {
                if $n == one(@$change) {
                    @toPrint[2*$n..2*$n+3] = ('  ',"\\/","/\\",'  ') Z~ @toPrint[2*$n..2*$n+3];	
                    $n += 2;
                } else {
                    @toPrint[2*$n,2*$n+1] = ('__','  ') Z~ @toPrint[2*$n,2*$n+1];
                    $n++;
                }
            }
            $input = $tmpArr;	
        }
    }
    pop @toPrint;
    say join("\n", @toPrint);





    sub sort_1_step ($input) {
        my $n = $input.end-1;
        my $output = [ '-1' xx $input.elems ];
        my $change = [];
        while $n >= 0 {
            if $input[$n] cmp $input[$n+1] ne 'Decrease' {
                # no flip
                $output[$n+1] = $input[$n+1];
                $n--;
            } else {
                # flip $n and $n+1
                $output[$n,$n+1] = $input[$n+1,$n];
                push @$change,$n;
                $n -= 2;
            }
        }
        $output[0] = $input[0] if $output[0] < 0;
        return ($output,$change);
    }

## Correctness

This program is correct. It builds the output as an array of lines, and
calculates each "step" as a column (right to left) which it fills in from top
to bottom, skipping two lines if the current line was swapped with the next.

The program prefers to make its flips as far right as possible.

It also doesn't score fully on solution elegance. This solution is one column
wider than it has to be:

    $ perl6 vvoorr 0816245397
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

Though the program author has made a number of choices I don't agree with, he's
*consistent* in making them.

## Clarity of intent

The program is fairly easy to read except for two things. The first is the
un-idiomatic things listed in the next section.

The second is the three very long lines responsible for collecting all the
output in the main loop. These lines are arguably very important, but they are
a combination of array slices, zipping metaops, and (in one case) a statement
modifier `for` loop. This makes them cognitively heavy, and hard to decipher.

## Idiomatic use of Perl 6

Encapsulating something into a scalar array, only to "deref" it into a list on
the next line:

    my $input = [ @*ARGS.split('') ];
    my @toPrint = (('_ '~$_,'   ') for @$input);

This happens several times in the program. Sometimes the dereferences are not
even necessary, like before pushing to an array:

    push @$change,$n;

I think we can safely say that this person has done some Perl 5 before, and
can't quite let go. `;-)`

And this is the most flamboyant way I've seen to do numeric comparison:

    if $input[$n] cmp $input[$n+1] ne 'Decrease' {

(Better written `if $input[$n] <= $input[$n+1]`. Note how the unfortunate
reversal of `Increase` and `Decrease` makes the expression as written extra
befuddling.)

## Brevity

The program is delightfully short, although the lines are long and often cryptic.
