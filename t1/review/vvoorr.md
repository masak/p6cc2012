    use v6;
    my $inputFile = 'input';

    my $statements;				# format: $statements{$WHO}{$WHAT} = 1
    my %number_of_statements;	# number of statements asserted by everyone
    my %people;					# contain all the islanders, no matter they say anything or not
    my %peopleMentionedBy;		# contain all the islanders, and who mentioned him

    # read the input 
    my $input = chomp slurp($inputFile);
    #$input = chomp $input;
    exit unless $input;

    # record all the statements, and who mentioned who
    for $input.split(/\n/) -> $line {
        unless $line ~~ m/^\w+\:\s+\w+/ {
            say "Lines must be on the form '<name>: <utterance>'";
            exit;
        }

        my @words = $line.words;
        last unless +@words;
        my $who = shift @words;
        $who = substr($who,0,*-1);
        my $said = @words.join(' ');
        my @tmp = parse_statement($said);
        
        %number_of_statements{$who}++;
        $statements{$who}{$said} = 1;
        %people{$who} = 1;
        given @tmp[0] {
            when 'type1' {%people{@tmp[1]} = 1;%peopleMentionedBy{@tmp[1]}.push($who);}
            when 'type2' {%people{@tmp[1]} = 1;%peopleMentionedBy{@tmp[1]}.push($who);}
            when 'type3' {%people{@tmp[1]} = 1; %people{@tmp[2]} = 1; %peopleMentionedBy{@tmp[1]}.push($who); %peopleMentionedBy{@tmp[2]}.push($who);}
            when 'type4' {%people{@tmp[1]} = 1; %people{@tmp[2]} = 1; %peopleMentionedBy{@tmp[1]}.push($who); %peopleMentionedBy{@tmp[2]}.push($who);}
        }
    }
    exit if $statements.elems == 0;

    # find out who is mentioned but didn't speak
    my %mentioned := set %peopleMentionedBy.keys;
    my %spoken := set $statements.keys;
    my %dumb := %mentioned (-) %spoken;
    if %dumb.elems {
        for %dumb.keys -> $dumb {
            for %peopleMentionedBy{$dumb}.kv -> $k,$v {
                say "$v mentions $dumb but $dumb doesn't say anything." ;
            }
        }
        exit;
    }



    # prepare for iteration
    my @iterate = [X] ([1,0] xx %people.elems);

    my @curState = 0 xx %people.elems;
    my @peopleHadSpoken = $statements.keys;
    my $numberOfSolution = 0;
    my @solution;

    # iterate over every possible state
    while @iterate.elems > 0 {
        @curState[$_] = shift @iterate for 0..%people.end;
        # when @curState is (0,1), we try ('knave','knight') for the two guys in @people
        
        my $counter = 0;
        my %peopleState;
        %peopleState.push($_,(@curState[$counter++] ?? 'knight' !! 'knave')) for %people.keys;
    ####say (%peopleState.keys,%peopleState.values);
        
        # evaluate the correctness of each statement
        my %number_of_correct_statements;
        %number_of_correct_statements{$_} = 0 for @peopleHadSpoken;
        for $statements.keys -> $name {
            for $statements{$name}.keys -> $statement {
    ####			my $a = state_is_true($statement,%peopleState);
                %number_of_correct_statements{$name} += state_is_true($statement,%peopleState);
    ####say ($a,%number_of_correct_statements{$name},$statement);
            }
        }
    ####say (%number_of_correct_statements.keys,%number_of_correct_statements.values);
    ####say (%number_of_statements.keys,%number_of_statements.values);


        # evaluate the current knight / knave state
        my @numberCorrectStatements = %number_of_correct_statements{@peopleHadSpoken};
        my @expectedNumberCorrectStatements <== map {%peopleState{$_} eq 'knight' ?? %number_of_statements{$_} !! 0} <== @peopleHadSpoken;
        #my @expectedNumberCorrectStatements;
        #push @expectedNumberCorrectStatements,(%peopleState{$_} eq 'knight' ?? %number_of_statements{$_} !! 0) for @peopleHadSpoken;
        my $deviation = [+] (@numberCorrectStatements >>ne<< @expectedNumberCorrectStatements);

        if $deviation == 0 {
    #		say "({%peopleState.keys}) == ({%peopleState.values})";
            push @solution,{'names'=>[%peopleState.keys],'types'=>[%peopleState.values]};
            $numberOfSolution++;
        }
    }
    unless $numberOfSolution {
        say "No solutions." ;
        exit;
    }
    say "Many solutions:" if $numberOfSolution > 1;
    for @solution -> $soln {
        my @toPrint = ("$soln<names>[$_]: $soln<types>[$_]" for 0..$soln<names>.end);
        say @toPrint.join('; ');
    }










    sub state_is_true($statement,%peopleState){
        my @tmp = parse_statement($statement);
        given @tmp[0] {
            when one('type1','type2') {return %peopleState{@tmp[1]} eq @tmp[2] ?? 1 !! 0;}
            when 'type3' {return %peopleState{@tmp[1]} eq %peopleState{@tmp[2]} ?? 1 !! 0;}
            when 'type4' {return %peopleState{@tmp[1]} ne %peopleState{@tmp[2]} ?? 1 !! 0;}
        };
    }

    sub parse_statement($line){
        my @words = $line.words;
        if @words.elems < 4 {
            say "Unrecognized utterance \'$line\'";
            exit;
        }

        if @words[3] ~~ m/knight/ {
            return ('type1',@words[0],'knight');
        } elsif @words[3] ~~ m/knave/ {
            return ('type2',@words[0],'knave');
        } elsif (@words.end >= 6) && (@words[6] ~~ m/same/) {
            return ('type3',@words[0],@words[2]);
        } elsif (@words.end >= 5) && (@words[5] ~~ m/different/) {
            return ('type4',@words[0],@words[2]);
        } else {
            say "Unrecognized utterance \'$line\'";
            exit;
        }
    }

## Correctness

The program appears correct.

## Consistency

The contestant makes a few non-standard decisions in terms of commented-out
code and line length. The former could be seen as either a careless residue or
a useful bit of documentation. The latter could be seen as either annoying or
endearing.

Worse are the lines which are *not* debug statements but that were commented
out, without a reason given for why. That makes this work of art look a bit
unfinished.

## Clarity of intent

Not sure `type1`, `type2`, `type3`, and `type4` conveys the maximal amount of
information to the interested reader.

I chuckle at the shortcuts taken in `parse_statement`. That's thinking outside
of the box, at least.

## Algorithmic efficiency

`my @iterate = [X] ([1,0] xx %people.elems);` prepares a universe of all
possible combinations.

## Idiomatic use of Perl 6

Interesting use of set difference of hashes in `my %dumb := %mentioned (-)
%spoken;`.

## Brevity

Fairly short and to the point.
