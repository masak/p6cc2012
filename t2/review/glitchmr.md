    #!/usr/bin/env perl6
    my @syllables = <5 7 5>;
    # Load the dictionary and get number of syllables in probably not
    # working well way (English is so difficult).
    my %dict = 'dict'.IO.lines.classify: { +.comb: / <[aeiouy]>+ / };
    my @results;
    for @syllables {
        my @words;
        my $syllables-left = $_;
        while $syllables-left {
            my $number-of-syllables = (1 .. ($syllables-left min 6)).pick;
            $syllables-left -= $number-of-syllables;
            @words.push: %dict{$number-of-syllables}.pick;
        }
        # I initially wanted to completely remove debugging code, but
        # actually debugging output looks nicely, so feel free to uncomment
        # it.
        
        # say "# {@(@results, $(@words)).perl}";
        
        # 15 is value I've got when experimenting with it
        redo if @words.chars < 15 || @results && @results[*-1].chars != @words.chars;
        @results.push: $(@words);
    }
    .say for @results;

## Correctness

The algorithm conflates "syllables" with "vowels", which doesn't work all that
well in English. To take just one example of many, this sentence alone contains
six words in which the final "e" is silent and doesn't count as a syllable.

Which is a pity, because that looks totally salvageable. Apart from that, it's
a fast, dependable solution.

## Consistency

No complaint here. Not one.

## Clarity of intent

Among all the solutions, this is the one I can take all at a time, and feel
comfortable that I know all that's going on. That's worth a lot.

The comments all carry their own weight, too.

There's actually two magic numbers in here, 6 and 15. The 15 is explained as
being the result of trial-and-error. Fair enough. The 6 sets an upper limit
to the number of syllables in a word. Not really clear why.

## Algorithmic efficiency

One of the fastest solutions. Usually runs in two or three seconds on my laptop.

That it's fast is slightly surprising, because it *commits* to one haiku width
early, and then keeps throwing out lines that don't conform. Well, that shows
how simple outruns clever in some cases, I guess.

## Idiomatic use of Perl 6

Very nice use of `.classify`.

Hm, if it were me, I would've promoted `my $syllables-left = $_;` to be a
parameter to the `for` loop. The only reason it can't be done straight off is
that we keep decrementing that variable, so we'd need to add `is copy`, too.

## Brevity

Oh yes! If you want brevity, this is *it*.

## External resources

    abs
    accepts
    accessed
    action
    adverb
    after
    all
    allowed
    any
    arity
    array
    associative
    attribute
    base
    bless
    block
    bounds
    call
    callable
    candidates
    caps
    capture
    carl masak
    category
    ceiling
    changed
    char
    chars
    chomp
    chop
    chunks
    classify
    clone
    code
    column
    comb
    comment
    complex
    composer
    condition
    conj
    constraints
    construct
    contents
    cool
    copy
    count
    create
    cursor
    date
    day
    declaration
    decode
    default
    defined
    denominator
    directive
    div
    duration
    eager
    enclosing
    encode
    end
    error
    exception
    exp
    expected
    fail
    failure
    features
    file
    filename
    first
    flat
    flip
    floor
    format
    from
    full
    gist
    got
    grammar
    grep
    handled
    hash
    how
    illegal
    index
    infinite
    infix
    instant
    instead
    int
    invert
    iterator
    join
    junction
    key
    keys
    larry wall
    length
    level
    line
    lines
    list
    localizer
    log
    macro
    make
    map
    match
    max
    message
    method
    min
    misplaced
    mode
    modified
    month
    mu
    multi
    name
    named
    new
    nil
    none
    norm
    nude
    numerator
    numeric
    of
    old
    one
    operation
    operator
    optional
    orig
    package
    packages
    pair
    pairs
    parameter
    parcel
    parent
    parse
    parts
    path
    payload
    perl
    perl six
    phaser
    pick
    placeholder
    plus
    polar
    pop
    positional
    print
    private
    prompt
    push
    radix
    rand
    range
    rat
    rational
    re
    real
    reason
    reduce
    regex
    replacement
    reserved
    reverse
    right
    role
    roll
    roots
    rotate
    round
    routine
    say
    scope
    shift
    sign
    signature
    sort
    source
    splice
    split
    strangely consistent
    stringy
    sub
    symbol
    target
    thing
    throw
    to
    today
    trim
    truncate
    twigil
    type
    unshift
    unwrap
    value
    values
    variable
    version
    what
    whatever
    when
    which
    words
    wrap
    year
