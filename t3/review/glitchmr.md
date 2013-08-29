    #!/usr/bin/env perl6
    # t3 was so annoying. I even have put @problems variable because of that.

    enum Wire <Straight Up Down None>;
    my @targets = @*ARGS[0].comb;
    my @wires = [] xx @targets;

    # Now you have 99 problems.
    my @problems;

    sub draw-wire($start, $end, @wires) {
        my $order = 0 <=> $end - $start;
        my $direction = $order == Increase ?? Up !! Down;
        my $i = -1;

        # Statement that loads your problems.
        my $problems = @problems[$end];

        # You have a problem when you're at the end of wires
        $problems = 1 if @wires.end == $end;

        # Yes, I'm iterating from end to start.
        for $end ...^ $start {
            $i++;
            my $pos = $_ + ($direction == Down);
            @wires[$pos][$i] ||= any();
            if $problems {
                if @wires[$pos][$i] == Up | Down {
                    $problems--;
                }
                else {
                    redo;
                }
            }
            @wires[$pos][$i] |= $direction;
            $_ = None if !$_ || $_ == Straight given @wires[$_ + $order][$i];
            @problems[$pos - 1]++;
        }
    }
    # First, parse longest differences.
    for reverse sort { abs $_ - @targets[$_] }, @targets.keys {
        draw-wire $_, @targets[$_], @wires;
    }

    my $max-wire = max @wires.map: +*;

    # Render the wires
    for @wires.kv -> $id, @wire {
        my @picture = @wire[^$max-wire].map: {$_ || Straight};
        # Print first line only if the first ID isn't 0.
        if $id {
            print q[ ] x 3;
            for @picture {
                # Would use ｢｣, but Rakudo doesn't appear to support those.
                when Up & Down { print Q[\/] }
                when Up        { print Q[ /] }
                when      Down { print Q[\ ] }
                default        { print Q[  ] }
            }
            print "\n";
        }
        printf "%-2i_", $id;
        for @picture {
            when Up & Down { print Q[/\] }
            when Up        { print Q[/ ] }
            when      Down { print Q[ \] }
            when Straight  { print Q[__] }
            default        { print Q[  ] }
        }
        say "_ @targets[$id]";
    }

## Correctness

This program is not correct.

No, scratch that. I'm rather impressed at how not correct this program is. From
the source code, it doesn't look so bad.

An example might help.

    $ perl6 glitchmr 6319845270
    0 _________________________________  _ 6
       \                                /
    1 _ \____________________________  / _ 3
         \                            /  
    2 ___ \  ______________________  / ___ 1
           \/                       /    
    3 _____/\____________________  / _____ 9
             \                    /      
    4 _______ \________________  / _______ 8
               \                /        
    5 _________ \  __________  / _________ 4
                 \/           /          
    6 ___________/\________  / ___________ 5
                   \        /            
    7 _____________ \  __  / _____________ 2
                     \/   /              
    8 _______________/\  / _______________ 7
                       \/                
    9 _________________/\_________________ 0

Observe how it connects the wrong numbers, and commits a number of "forbidden"
operations on the wires in doing so.

On the input `1087325496`, the program *hangs*. Also on inputs `1235864790`,
`4078369521`, `4725869130`, `5721380649`, and likely many others. I don't know
why or where it hangs. On the inputs where it doesn't hang, it's still kinda
slow.

The program seems to suffer from a lack of overall strategy, as evidenced by
the initial comment `# t3 was so annoying`.

## Consistency

`@wires` is used *both* as a global variable and a parameter to `draw-wires`.
But it's always the former being sent to the latter, so we might as well have
skipped the parameter.

There's something that feels inconsistent about having both that `None` enum
*and* using junctions to combine choices. Couldn't the lack of a choice simply
have been represented as `all()`, the empty conjunction?

## Clarity of intent

As a stylistic point, I think it's never a good idea to lean on the fact that
`True` implicitly numifies to `1`:

    my $pos = $_ + ($direction == Down);

Randall Schwartz once pointed out that if he were the BDFL, he'd make `True`
numify to 42 just to avoid people writing code like the above.

## Idiomatic use of Perl 6

This is the first time I've seen someone store conjunctions of enums in an
array, to later switch on them. That's innovative. Let it serve as a reminder
that statically optimizing `given` constructs in Perl 6 is probably a fool's
errand.

A small suggestion:

    # Yes, I'm iterating from end to start.
    for $end ...^ $start {

If you write it like this instead, you won't even need the comment:

    for reverse $start ^.. $end {

And it's somehow cognitively easier on the brain, at least I find it so.

## Brevity

It's short and to the point. Too bad about the correctness, though.
