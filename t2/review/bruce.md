    #! /usr/bin/env perl6
    
    use v6;
    use File::Find;
    
    sub haiku () {
    	my @word-sources = $*word-source.defined
    		?? $*word-source
    		!! find(dir => 'word-lists').grep(/csv$/).pick(*);
    	repeat {
    		my %*words = load-words(@word-sources.pop);
    		my @haiku = mad-lib;
    		return @haiku if +@haiku;
    	} while +@word-sources;
    	return [
    		"some hopeful outlook",
    		"recursion impossible",
    		"perplexed by failure"
    	];
    }
    
    sub mad-lib () {
    	my @templates = $*mad-lib.defined
    		?? $*mad-lib
    		!! find(dir => 'mad-libs').grep(/csv$/).pick(*);
    	repeat {
    		my @haiku = make-rectangular(@templates.pop);
    		return @haiku if +@haiku;
    	} while +@templates;
    	return;
    }
    
    sub make-rectangular($madlib) {
    	say "Mad-lib $madlib" if $*v;
    	my @template = load-template($madlib);
    	my @lengths = (20 .. 40).pick(*);
    	repeat {
    		my $*length = @lengths.pop;
    		my @haiku <== pack-lines() <== (5,7,5) Z @template;
    		return @haiku if +@haiku;
    	} while +@lengths;
    	return;
    }
    
    sub pack-lines (@lines) {
    	my @haiku;
    	for @lines -> $syllables, @speech-parts {
    		my $length-without-spaces = $*length - +@speech-parts +1;
    		my $packed = pack-syllables($length-without-spaces, $syllables, @speech-parts).join(' ');
    		if $packed.chars == $*length {
    			@haiku.push($packed);
    		}
    		else {
    			return;
    		}
    	}
    	return @haiku;
    }
    
    sub pack-syllables($length, $syllables, @speech-parts) {
    	my ($part, @remaining-parts) = @speech-parts;
    	my $words-by-syllables = %*words{$part};
    	my $max = $syllables - +@remaining-parts;
    
    	my @syllables = +@remaining-parts
    		?? (1 .. $max).pick(*)
    		!! $max;
    	repeat {
    		my $syllables-to-use = @syllables.pop;
    
    		next unless $words-by-syllables[$syllables-to-use].defined;
    		my @next-words = pack-letters($length, $syllables-$syllables-to-use, @remaining-parts, $words-by-syllables[$syllables-to-use]);
    		return @next-words if +@next-words;
    	} while +@syllables;
    	return;
    }
    
    sub pack-letters($length, $syllables, @remaining-speech-parts, $words-by-length) {
    	return if $length < 1;
    
    	if +@remaining-speech-parts {
    		my @lengths = $words-by-length.pairs.grep({ .value.defined && +(.value) }).pick(*);
    		repeat {
    			my ($len, $words) = @lengths.pop.kv;
    			my @next-words = pack-syllables($length-$len, $syllables, @remaining-speech-parts);
    			return ($words.pick, @next-words) if +@next-words;
    		} while +@lengths;
    	}
    	else {
    		return unless $words-by-length[$length].defined;
    		return $words-by-length[$length].pick;
    	}
    	return;
    }
    
    sub load-words($source-file) {
    	my %part;
    	my $count = 0;
    
    	say "Source $source-file" if $*v;
    	for open($source-file).lines -> $l {
    		my ($word, $syllables, $part-of-speech) = $l.split(',');
    		%part{$part-of-speech}[$syllables][$word.chars].push($word);
    		$count++;
    	}
    	say "Loaded $count words" if $*v;
    	return %part;
    }
    
    sub load-template($madlib-file) {
    	my @madlib;
    	for open($madlib-file).lines -> $l {
    		@madlib.push([$l.split(',')]);
    	}
    	return @madlib;
    }
    
    multi sub MAIN(Bool :$*v?, Str :$*mad-lib?, Str :$*word-source?) {
    	eager haiku.map: { .say };
    }

## Correctness

The program has the structure of a waterfall of subroutines, each one calling
the next, with backtracking happening naturally when a subroutine fails to find
something. Looks correct to me.

As to syllable counts in the word files, I don't agree with all of those.
Here's a handful I don't agree with:

    encyclopedia,5,noun
    about,1,adverb
    uses,1,abbreviation
    idea,1,noun
    juxtaposed,1,adjective

## Consistency

Nothing much to complain about here. In fact, it's so squeaky-clean,
consistency-wise, that I can just point out that I noticed that sometimes in
sub declarations, there's a space after the sub name and before the parens, and
sometimes there ain't.

## Clarity of intent

I love the effort that must have gone into making the *error message* in the
`haiku` sub... a rectangular haiku.

## Algorithmic efficiency

There's certainly a lot of searching going on here. The script appears to try a
bunch of different line lengths in random order, and simply throw away all but
the length it's looking for right now. That feels slightly wasteful compared to
some other solutions.

## Idiomatic use of Perl 6

Interesting use of dynamic variables. I'm not 100% sure I *like* it &mdash;
seems to me it creates a lot of coupling between subs &mdash; but it's
interesting nevertheless.

I find the use of pipes on line 39 very cute, and oddly appropriate. That same
line happens to also have a nice use of `infix:<Z>`.

## Brevity

Not super-short, no. But not long either.

## External resources

[`File::Find`](https://github.com/tadzik/perl6-File-Tools/blob/master/lib/File/Find.pm).

The seven `mad-libs/` files each contain "recipes" for making a haiku out of
parts of speech:

    noun,preposition,noun
    indefinite article,noun,verb,preposition,pronoun,verb,noun
    adjective,noun,adjective,preposition,noun

    adjective,noun,noun
    definite article,adjective,noun,verb
    preposition,indefinite article,noun,preposition

    verb,adverb,preposition,preposition,pronoun
    conjunction,adverb,transitive verb,pronoun,verb
    interjection,adjective,noun

    adjective,adjective,noun
    preposition,verb,conjunction,preposition,definite article,noun
    indefinite article,noun,noun

    preposition,adjective,noun
    preposition,noun,noun,verb
    pronoun,adjective,noun

    definite article,adjective,noun,verb
    conjuction,adverb,preposition,noun,noun
    indefinite article,adjective,noun,preposition,noun

    noun,adverb,verb
    adjective,noun,preposition,verb
    noun,transitive verb,noun

And the `word-lists` files contain lots of words:

`all-words.csv`:

    from,1,preposition
    the,1,definite article
    free,1,adjective
    encyclopedia,5,noun
    jump,1,verb
    to,1,preposition
    navigation,4,noun
    article,3,noun
    about,1,adverb
    japanese,3,noun
    poetic,3,adjective
    form,1,noun
    for,1,preposition
    haiku,2,noun
    poetry,3,noun
    write,1,verb
    in,1,preposition
    english,2,adjective
    see,1,verb
    other,2,adjective
    uses,1,abbreviation
    disambiguate,5,transitive verb
    verse,1,noun
    listen,2,verb
    help,1,verb
    info,2,noun
    separate,3,verb
    plural,2,adjective
    a,1,noun
    very,1,adjective
    short,1,adjective
    of,1,preposition
    typically,4,adverb
    by,1,preposition
    three,1,noun
    quality,3,noun
    essence,2,noun
    cutting,2,noun
    this,1,pronoun
    often,2,adverb
    represent,3,verb
    juxtaposition,5,noun
    two,1,adjective
    image,2,noun
    or,1,conjunction
    idea,1,noun
    and,1,conjunction
    word,1,noun
    between,2,preposition
    them,1,pronoun
    kind,1,noun
    verbal,2,adjective
    punctuation,4,noun
    mark,1,noun
    which,1,adjective
    signal,2,noun
    moment,2,noun
    separation,4,noun
    manner,2,noun
    juxtaposed,1,adjective
    element,3,noun
    related,1,adjective
    tradition,3,noun
    consist,2,intransitive verb
    on,1,preposition
    also,2,adverb
    known,1,adjective
    as,1,adverb
    mora,2,noun
    phrase,1,noun
    respectively,4,adverb
    any,1,adjective
    one,1,adjective
    may,1,verbal auxiliary
    end,1,noun
    with,1,preposition
    although,2,conjunction
    stated,1,adjective
    have,1,verb
    syllable,3,noun
    inaccurate,4,adjective
    not,1,adverb
    same,1,adjective
    seasonal,3,adjective
    reference,3,noun
    usual,2,adjective
    an,1,indefinite article
    extensive,3,adjective
    but,1,conjunction
    define,2,verb
    list,1,verb
    such,1,adjective
    modern,2,adjective
    increasingly,4,adverb
    unlikely,3,adjective
    follow,2,verb
    take,1,verb
    nature,2,noun
    their,1,adjective
    subject,2,noun
    use,1,noun
    continue,3,verb
    be,1,verb
    both,1,pronoun, plural in construction
    there,1,adverb
    common,2,adjective
    relatively,4,adverb
    recent,2,adjective
    perception,3,noun
    that,1,pronoun
    must,1,verb
    directly,3,adverb
    observe,2,verb
    everyday,3,adjective
    object,2,noun
    occurrence,3,noun
    print,1,noun
    single,2,adjective
    vertical,3,adjective
    line,1,noun
    while,1,noun
    appear,2,intransitive verb
    parallel,3,adjective
    previous,3,adjective
    hokku,2,noun
    given,2,adjective
    its,1,adjective
    current,2,adjective
    name,1,noun
    writer,2,noun
    at,1,preposition
    th,1,abbreviation
    century,3,noun
    content,2,adjective
    example,3,noun
    origin,3,noun
    development,4,noun
    bash,1,verb
    movement,2,noun
    west,1,adverb
    henderson,3,biographical name
    contemporary,4,adjective
    language,2,noun
    worldwide,2,adjective
    famous,2,adjective
    period,3,noun
    later,1,adverb
    bibliography,5,noun
    external,3,adjective
    links,1,noun plural
    edit,2,transitive verb
    prosody,3,noun
    contrast,2,verb
    characterize,4,transitive verb
    meter,2,noun
    count,1,verb
    sound,1,adjective
    unit,1,noun
    five,1,noun
    seven,2,noun
    among,1,preposition
    poem,2,noun
    fixed,1,adjective
    pattern,2,noun
    do,1,verb
    below,2,adverb
    illustrate,3,verb
    masters,2,adjective
    always,2,adverb
    constrain,2,transitive verb
    sometimes,2,adverb
    translate,2,verb
    elongate,2,verb
    vowel,2,noun
    diphthong,2,noun
    double,2,adjective
    consonant,3,adjective
    thus,1,adverb
    though,1,conjunction
    four,1,noun
    ha,1,interjection
    i,1,noun
    bu,1,abbreviation
    itself,2,pronoun
    speaker,2,noun
    would,1,verb
    view,1,noun
    comprise,2,transitive verb
    nasal,2,noun
    contain,2,verb
    only,2,adjective
    converse,2,noun
    some,1,adjective
    can,1,verb
    perceive,2,transitive verb
    symbol,2,noun
    used,1,adjective
    refer,2,verb
    long,1,adjective
    each,1,adjective
    correspond,3,intransitive verb
    kana,2,noun
    character,3,noun
    digraph,2,noun
    hence,1,adverb
    society,4,noun
    america,3,geographical name
    noted,1,adjective
    norm,1,noun
    they,1,pronoun, plural in construction
    trend,1,intransitive verb
    toward,2,adjective
    approximate,4,adjective
    duration,3,noun
    symbolize,3,verb
    imply,2,transitive verb
    season,2,noun
    metonym,3,noun
    citation,3,noun
    need,1,noun
    difficult,3,adjective
    who,1,pronoun
    lack,1,verb
    cultural,3,adjective
    spot,1,noun
    include,2,transitive verb
    frog,1,noun
    spring,1,noun
    rain,1,noun
    shower,2,noun
    late,1,adjective
    autumn,2,noun
    early,2,adverb
    winter,2,noun
    fill,1,verb
    role,1,noun
    somewhat,2,pronoun
    analogous,3,adjective
    caesura,3,noun
    classical,3,adjective
    western,2,adjective
    volta,2,biographical name
    sonnet,2,noun
    depend,2,intransitive verb
    chosen,2,noun
    position,3,noun
    within,2,adverb
    it,1,pronoun
    briefly,2,adverb
    cut,1,verb
    stream,1,noun
    suggest,2,transitive verb
    preceding,3,adjective
    following,3,adjective
    provide,2,verb
    dignified,3,adjective
    ending,2,noun
    conclude,2,verb
    heighten,2,verb
    sense,1,noun
    closure,2,noun
    fundamental,4,adjective
    aesthetic,3,adjective
    internal,3,adjective
    sufficient,3,adjective
    independent,4,adjective
    context,2,noun
    will,1,verb
    bear,1,noun
    consideration,4,noun
    complete,2,adjective
    work,1,noun
    lend,1,verb
    structural,3,adjective
    support,2,transitive verb
    allow,2,verb
    stand,1,verb
    distinguish,3,verb
    second,2,adjective
    subsequent,3,adjective
    employ,2,transitive verb
    semantic,3,adjective
    syntactic,3,adjective
    disjuncture,3,noun
    even,1,noun
    point,1,noun
    occasionally,5,adverb
    stop,1,verb
    sh,1,interjection
    sentence,2,noun
    particle,3,noun
    generally,4,adverb
    since,1,adverb
    direct,2,verb
    equivalent,3,adjective
    poet,2,noun
    dash,1,verb
    ellipsis,3,noun
    break,1,verb
    create,2,verb
    intended,1,adjective
    prompt,1,transitive verb
    reader,2,noun
    reflect,2,verb
    relationship,4,noun
    part,1,noun
    old,1,adjective
    pond,1,noun
    wind,1,noun
    mt,1,abbreviation
    fuji,2,noun
    ya,1,abbreviation
    neither,2,conjunction
    remain,2,intransitive verb
    nor,1,conjunction
    balance,2,noun
    fragment,2,noun
    first,1,adjective
    against,1,preposition
    apparent,3,adjective
    translation,3,noun
    mean,1,verb
    edo,1,geographical name
    best,1,adjective
    transliterate,4,transitive verb
    into,2,preposition
    hiragana,4,noun
    ru,1,symbol
    ka,1,abbreviation
    wa,1,abbreviation
    bi,1,noun or adjective
    ko,1,noun
    mu,1,noun
    mi,1,noun
    leap,1,verb
    another,3,adjective
    mo,1,abbreviation
    wo,1,abbreviation
    gu,1,abbreviation
    re,1,noun
    sa,1,abbreviation
    ho,1,interjection
    ge,1,abbreviation
    na,1,symbol
    ri,1,abbreviation
    cold,1,adjective
    monkey,2,noun
    seem,1,intransitive verb
    little,2,adjective
    coat,1,noun
    straw,1,noun
    he,1,pronoun
    treat,1,verb
    gi,1,abbreviation
    ni,1,symbol
    se,1,symbol
    te,1,symbol
    my,1,adjective
    fan,1,noun
    gift,1,noun
    equate,1,verb
    ame,1,abbreviation
    me,1,pronoun
    go,1,verb
    da,1,abbreviation
    how,1,adverb
    many,1,adjective
    you,1,pronoun
    drink,1,verb
    cuckoo,2,noun
    opening,2,noun
    stanza,2,noun
    orthodox,3,adjective
    collaborate,4,intransitive verb
    linked,1,adjective
    derivative,4,noun
    time,1,noun
    matsuo,2,biographical name
    begin,2,verb
    incorporated,1,adjective
    combination,4,noun
    prose,1,noun
    painting,2,noun
    latter,2,adjective
    term,1,noun
    now,1,adverb
    applied,2,adjective
    retrospective,4,adjective
    all,1,adjective
    when,1,adverb
    describe,2,transitive verb
    considered,3,adjective
    obsolete,3,adjective
    main,1,noun
    elevated,4,adjective
    new,1,adjective
    popularity,5,noun
    made,1,adjective
    most,1,adjective
    important,3,adjective
    setting,1,noun
    tone,1,noun
    whole,1,adjective
    composition,4,noun
    individual,5,adjective
    understood,3,adjective
    school,1,noun
    promote,2,transitive verb
    anthology,4,noun
    give,1,verb
    birth,1,noun
    what,1,pronoun
    his,1,adjective
    torque,1,noun
    sketch,1,noun
    travel,2,verb
    diary,3,noun
    sub,1,noun
    genre,1,noun
    narrow,2,adjective
    road,1,noun
    interior,4,adjective
    classic,2,adjective
    literature,4,noun
    deify,3,transitive verb
    imperial,4,adjective
    government,3,noun
    shinto,2,noun
    religious,3,adjective
    headquarters,3,noun plural but singular or plural in construction
    hundred,2,noun
    year,1,noun
    after,2,adverb
    death,1,noun
    because,2,conjunction
    raised,1,adjective
    playful,2,adjective
    game,1,noun
    wit,1,verb
    sublime,2,verb
    revere,2,transitive verb
    saint,1,noun
    japan,2,adjective
    familiar,3,noun
    throughout,2,adverb
    world,1,noun
    grave,1,transitive verb
    next,1,adjective
    style,1,noun
    arise,1,intransitive verb
    kit,1,noun
    era,1,noun
    attempt,2,transitive verb
    revive,2,verb
    value,2,noun
    rescue,2,transitive verb
    stultify,3,transitive verb
    condition,3,noun
    day,1,noun
    great,1,adjective
    where,1,adverb
    combined,2,noun
    affection,3,noun
    painterly,3,adjective
    kobayashi,4,biographical name
    popular,3,adjective
    however,3,conjunction
    individualist,6,noun
    humanism,3,noun
    approach,2,verb
    writing,1,noun
    demonstrate,3,verb
    whose,1,adjective
    miserable,4,adjective
    childhood,2,noun
    poverty,3,noun
    sad,1,adjective
    life,1,noun
    devotion,3,noun
    pure,1,adjective
    land,1,noun
    sect,1,noun
    buddhism,2,noun
    evident,3,adjective
    immediately,5,adverb
    accessible,4,adjective
    wide,1,adjective
    audience,3,noun
    reformer,3,noun
    modernize,3,verb
    prolific,3,adjective
    chronic,2,adjective
    ill,1,adjective
    during,2,preposition
    significant,4,adjective
    dislike,2,noun
    stereotype,3,transitive verb
    deprecatory,5,adjective
    meaning,2,noun
    monthly,2,adverb
    twice,1,adverb
    gathering,3,noun
    regard,2,noun
    trite,1,adjective
    hackneyed,2,adjective
    criticize,3,verb
    like,1,verb
    intellectual,5,adjective
    general,3,adjective
    strong,1,adjective
    influence,3,noun
    culture,2,noun
    favored,2,adjective
    particularly,5,adverb
    european,4,adjective
    concept,2,noun
    air,1,noun
    adapt,1,verb
    literally,4,adverb
    popularize,4,verb
    column,2,noun
    essay,2,transitive verb
    newspaper,3,noun
    up,1,adverb
    formal,2,adjective
    being,2,noun
    agnostic,3,noun
    further,2,adverb
    discard,2,verb
    propose,2,verb
    abbreviation,5,noun
    predate,2,transitive verb
    then,1,adverb
    date,1,noun
    revisionism,4,noun
    deal,1,noun
    severe,2,adjective
    blow,1,verb
    survive,2,verb
    chiefly,2,adverb
    original,3,noun
    rarely,2,adverb
    before,2,adverb or adjective
    autobiography,6,noun
    journal,2,noun
    base,1,noun
    today,2,adverb
    artist,2,noun
    combine,2,verb
    photograph,3,noun
    carving,2,noun
    natural,3,adjective
    stone,1,noun
    make,1,verb
    monument,3,noun
    practice,2,verb
    city,1,noun
    matsuyama,4,geographical name
    more,1,adjective
    than,1,conjunction
    westerner,3,noun
    dutchman,2,noun
    dutch,1,adverb
    commissioner,4,noun
    trade,1,noun
    post,1,noun
    nagasaki,4,geographical name
    your,1,adjective
    arm,1,noun
    fast,1,adjective
    thunderbolt,3,noun
    pillow,2,noun
    journey,2,noun
    outside,2,noun
    imitate,3,transitive verb
    understanding,4,noun
    principle,3,noun
    scholar,2,noun
    basil,2,noun
    hall,1,noun
    chamberlain,3,noun
    william,2,biographical name
    george,1,noun
    aston,2,biographical name
    mostly,2,adverb
    dismiss,2,transitive verb
    advocate,3,noun
    noguchi,3,biographical name
    proposal,3,noun
    american,3,noun
    publish,2,verb
    magazine,3,noun
    february,3,noun
    brief,1,adjective
    outline,2,noun
    own,1,adjective
    effort,2,noun
    exhortation,4,noun
    pray,1,verb
    try,1,verb
    publishing,3,noun
    well,1,noun
    french,1,transitive verb
    france,1,biographical name
    introduce,3,transitive verb
    paul,1,noun
    louis,2,biographical name
    around,1,adverb
    read,1,verb
    imagism,3,noun
    theoretician,5,noun
    flint,1,noun
    pass,1,verb
    idiosyncrasy,6,noun
    member,2,noun
    club,1,noun
    ezra,2,noun
    pound,1,noun
    lowell,2,biographical name
    trip,1,verb
    london,2,biographical name
    meet,1,verb
    find,1,verb
    out,1,adverb
    she,1,pronoun
    return,2,verb
    united,1,adjective
    worked,1,adjective
    interest,3,noun
    considerable,4,adjective
    notably,3,adverb
    station,2,noun
    metro,2,noun
    notwithstanding,4,preposition
    several,3,adjective
    explain,2,verb
    spirit,2,noun
    yet,1,adverb
    history,3,noun
    spanish,2,noun
    mexican,3,noun
    nobel,2,biographical name
    prize,1,noun
    winner,2,noun
    paz,1,biographical name
    diplomat,3,noun
    horace,2,biographical name
    englishman,3,noun
    live,1,verb
    produce,2,verb
    series,2,noun
    zen,1,noun
    asian,1,adjective
    publication,4,noun
    volume,2,noun
    war,1,noun
    speaking,1,adjective
    study,1,noun
    major,2,adjective
    interpreter,4,noun
    stimulate,3,verb
    essential,3,adjective
    possibility,5,noun
    selected,3,adjective
    book,1,noun
    titled,1,adjective
    pepper,2,noun
    pod,1,noun
    together,3,adverb
    present,2,noun
    critical,3,adjective
    theory,3,noun
    add,1,verb
    comment,2,noun
    critic,2,noun
    apply,2,verb
    third,1,adjective
    rhyme,1,noun
    should,1,verbal auxiliary
    utilize,2,transitive verb
    resource,2,noun
    personal,3,adjective
    experience,4,noun
    motive,2,noun
    notion,2,noun
    resonate,3,verb
    north,1,adverb
    widely,2,adverb
    harold,2,biographical name
    introduction,4,noun
    doubleday,3,biographical name
    anchor,2,noun
    revision,3,noun
    bamboo,2,noun
    broom,1,noun
    mifflin,2,biographical name
    occupation,4,noun
    household,2,noun
    share,1,noun
    appreciation,5,noun
    bond,1,noun
    every,2,adjective
    tercet,2,noun
    whereas,2,conjunction
    never,2,adverb
    recognize,3,transitive verb
    seventeen,3,noun
    normal,2,adjective
    mode,1,noun
    accentual,4,adjective
    rather,2,adverb
    syllabic,3,adjective
    emphasize,3,transitive verb
    order,2,verb
    event,1,noun
    nevertheless,4,adverb
    concentrate,3,verb
    country,2,noun
    balkans,2,geographical name
    impossible,4,adjective
    format,2,noun
    matter,2,noun
    definitive,4,adjective
    fewer,2,pronoun, plural in construction
    indicate,3,transitive verb
    implicit,3,adjective
    compare,2,verb
    situation,4,noun
    focus,2,noun
    place,1,noun
    human,2,adjective
    consider,3,verb
    broad,1,adjective
    range,1,noun
    suitable,2,adjective
    urban,2,adjective
    loosen,2,verb
    standard,2,noun
    result,2,intransitive verb
    source,1,noun
    claim,1,transitive verb
    justify,3,verb
    blur,1,noun
    definition,4,noun
    boundary,2,noun
    st,1,abbreviation
    thriving,1,adjective
    community,4,noun
    mainly,2,adverb
    communicate,4,verb
    through,1,preposition
    national,3,adjective
    regional,3,adjective
    northern,2,adjective
    europe,2,geographical name
    sweden,2,geographical name
    germany,3,geographical name
    belgium,2,geographical name
    netherlands,3,geographical name
    central,2,adjective
    southeast,2,adverb
    croatia,3,geographical name
    slovenia,3,geographical name
    serbia,2,geographical name
    bulgaria,3,geographical name
    poland,2,geographical name
    romania,3,geographical name
    russia,2,geographical name
    asi,1,abbreviation
    laureate,3,noun
    tagore,2,biographical name
    composed,2,adjective
    bengali,2,noun
    gujarati,4,noun
    desai,2,biographical name
    festival,3,adjective
    bangalore,3,geographical name
    over,1,adverb
    bangladesh,3,geographical name
    us,1,pronoun
    asia,1,geographical name
    pakistan,3,geographical name
    omer,1,noun
    active,2,adjective
    global,2,adjective
    nuclear,3,adjective
    disarm,2,verb
    hiroshima,4,geographical name
    various,3,adjective
    peace,1,noun
    conference,3,noun
    uk,1,abbreviation
    group,1,noun
    international,5,adjective
    association,5,noun
    exchange,2,noun
    foreign,2,adjective
    president,3,noun
    council,2,noun
    herman,2,biographical name
    van,1,noun
    notable,3,adjective
    april,1,noun
    rwy,1,abbreviation
    predecessor,4,noun
    chinese,2,noun
    tamil,2,noun
    treasure,2,noun
    writings,2,noun plural
    tanka,2,noun
    redirect,3,transitive verb
    quote,1,verb
    simply,2,adverb
    john,1,noun
    uncut,2,adjective
    david,2,noun
    cup,1,noun
    tea,1,noun
    humanity,4,noun
    press,1,noun
    isbn,1,abbreviation
    trace,1,noun
    dream,1,noun
    landscape,2,noun
    memory,3,noun
    university,4,noun
    people,2,noun
    grieg,1,biographical name
    retrieve,2,verb
    beyond,2,adverb
    den,1,noun
    cor,1,noun
    nd,1,abbreviation
    edition,2,noun
    simon,2,noun
    december,3,noun
    ban,1,verb
    technique,2,noun
    net,1,noun
    http,1,abbreviation
    www,1,abbreviation
    criticism,3,noun
    html,1,noun
    richard,2,biographical name
    gilbert,2,noun
    stalk,1,noun
    wild,1,adjective
    columbia,3,noun
    note,1,transitive verb
    carter,2,biographical name
    vol,1,abbreviation
    karen,2,noun
    lewis,2,noun
    cook,1,noun
    higginson,3,biographical name
    handbook,2,noun
    thirty,2,noun
    commentary,3,noun
    wave,1,verb
    revise,2,noun
    ed,1,noun
    shoemaker,3,noun
    hoard,1,noun
    theme,1,noun
    notebook,2,noun
    archive,2,noun
    com,1,abbreviation
    search,1,verb
    gallon,2,noun
    deep,1,adjective
    penguin,2,noun
    thomas,2,noun
    guide,1,noun
    pp,1,abbreviation
    sequence,2,noun
    ross,1,biographical name
    bruce,1,biographical name
    earl,1,noun
    mine,1,adjective
    false,1,adjective
    start,1,verb
    again,1,adverb
    reflection,3,noun
    review,2,noun
    issue,2,noun
    march,1,noun
    org,1,abbreviation
    flanders,2,geographical name
    max,1,noun
    german,2,adjective
    leiden,2,geographical name
    oriental,3,adjective
    connection,3,noun
    brill,1,noun
    bob,1,verb
    project,2,noun
    en,1,noun
    literary,3,adjective
    we,1,pronoun, plural in construction
    pioneer,3,noun
    gain,1,noun
    selection,3,noun
    summer,2,noun
    special,2,adjective
    feature,2,noun
    telegraph,2,noun
    co,1,abbreviation
    news,1,noun plural but singular in construction
    eu,1,symbol
    launch,1,verb
    rn,1,symbol
    charter,2,noun
    fish,1,noun
    water,2,noun
    times,1,preposition
    online,2,adjective
    cid,1,abbreviation
    otc,1,abbreviation
    rss,1,noun
    penny,2,noun
    teach,1,verb
    beginning,3,noun
    tokyo,2,geographical name
    son,1,noun
    cole,1,noun
    key,1,noun
    master,2,noun
    ken,1,verb
    text,1,noun
    open,1,adjective
    directory,4,adjective
    guideline,2,noun
    index,2,noun
    title,2,noun
    category,4,noun
    statement,2,noun
    june,1,noun
    september,3,noun
    perl,1,biographical name
    skip,1,verb
    home,1,noun
    bug,1,noun
    track,1,noun
    documentation,5,noun
    get,1,verb
    star,1,noun
    release,2,verb
    maurice,2,biographical name
    behalf,2,noun
    team,1,noun
    happy,2,adjective
    announce,2,verb
    useful,2,adjective
    usable,2,adjective
    distribution,4,noun
    available,2,adjective
    download,2,noun
    section,2,noun
    window,2,noun
    version,2,noun
    area,2,noun
    shortly,2,adverb
    distinction,3,noun
    specific,3,adjective
    implement,3,noun
    compiler,3,noun
    parrot,2,noun
    virtual,3,adjective
    machine,2,noun
    module,2,noun
    collected,3,adjective
    parse,1,verb
    error,2,noun
    much,1,adjective
    improve,2,verb
    std,1,abbreviation
    parser,2,noun
    close,1,verb
    accurate,3,adjective
    information,4,noun
    keep,1,verb
    less,1,adjective
    serious,3,adjective
    better,2,adjective
    failure,2,noun
    junction,2,noun
    magnitude,3,noun
    texas,2,noun
    ascii,1,noun
    set,1,verb
    bag,1,noun
    operator,4,noun
    nest,1,noun
    pair,1,noun
    correct,2,transitive verb
    output,2,noun
    block,1,noun
    hash,1,transitive verb
    performance,3,noun
    improvement,3,noun
    fix,1,verb
    report,2,noun
    deprecate,3,transitive verb
    modify,3,verb
    due,1,adjective
    change,1,verb
    specification,5,noun
    removed,1,adjective
    loop,1,noun
    become,2,verb
    lazy,2,adjective
    evaluate,3,transitive verb
    eager,2,adjective
    sink,1,verb
    void,1,adjective
    if,1,conjunction
    last,1,verb
    routine,2,noun
    run,1,verb
    call,1,verb
    anymore,2,adverb
    unary,2,adjective
    hyper,2,adjective
    ops,1,noun
    descend,2,verb
    array,2,transitive verb
    level,2,noun
    map,1,noun
    str,1,abbreviation
    replace,2,transitive verb
    tc,1,symbol
    leading,2,adjective
    rule,1,noun
    under,2,adverb
    convert,2,verb
    exist,2,intransitive verb
    expect,2,verb
    conversion,3,noun
    front,1,noun
    meta,2,geographical name
    quantifier,4,noun
    capture,2,noun
    bind,1,verb
    slot,1,noun
    either,2,adjective
    zero,2,noun
    match,1,noun
    future,2,adjective
    nil,1,noun
    code,1,noun
    manage,2,verb
    transition,3,noun
    handle,2,noun
    appropriate,4,transitive verb
    upcoming,3,adjective
    quite,1,adverb
    advanced,2,adjective
    macro,2,adjective
    thread,1,noun
    concurrency,4,noun
    string,1,noun
    interactive,4,adjective
    understand,3,verb
    synopsis,3,noun
    missing,2,adjective
    tried,1,adjective
    smart,1,adjective
    enough,1,adjective
    inform,2,verb
    programmer,3,noun
    miss,1,verb
    broken,2,adjective
    welcome,2,transitive verb
    tutorial,4,adjective
    material,4,adjective
    document,3,noun
    draft,1,noun
    doc,1,noun
    pdf,1,noun
    thanks,1,noun plural
    contribute,3,verb
    sponsor,2,noun
    making,1,noun
    possible,3,adjective
    ask,1,verb
    mailing,2,noun
    join,1,verb
    entry,2,noun
    bookmark,2,noun
    closed,1,adjective
    site,1,noun
    weekly,2,adverb
    maven,2,noun
    gut,1,noun
    strange,1,adjective
    consistent,3,adjective
    de,1,abbreviation
    michael,2,noun
    october,3,noun
    august,2,adjective
    july,2,noun
    january,3,noun
    log,1,noun
    proud,1,adjective
    power,2,noun
    learn,1,verb
    involved,1,adjective
    quick,1,adjective
    mac,1,noun
    os,1,noun
    latest,1,adjective
    strawberry,3,noun
    platform,2,noun
    recommend,3,transitive verb
    stable,2,noun
    running,1,noun
    operating,1,adjective
    system,2,noun
    binary,3,noun
    already,2,adverb
    install,2,transitive verb
    probably,3,adverb
    type,1,noun
    command,2,verb
    simple,2,adjective
    way,1,noun
    look,1,verb
    app,1,noun
    compile,2,transitive verb
    terminal,3,adjective
    application,4,noun
    utility,3,noun
    folder,2,noun
    default,2,noun
    win,1,verb
    exactly,3,adverb
    everywhere,3,adverb
    else,1,adverb
    without,2,preposition
    package,2,noun
    developer,4,noun
    channel,2,noun
    access,2,noun
    browser,2,noun
    aim,1,verb
    come,1,verb
    padre,2,noun
    ide,1,abbreviation
    learning,1,noun
    contact,2,noun
    commercial,3,adjective
    offer,2,verb
    tip,1,verb
    good,1,adjective
    local,2,adjective
    think,1,verb
    blog,1,noun
    jobs,1,biographical name
    dev,1,abbreviation
    usa,1,abbreviation
    australia,3,geographical name
    brazil,2,geographical name
    kingdom,2,noun
    boston,2,noun
    chicago,3,geographical name
    dallas,2,biographical name
    los,1,abbreviation
    york,1,biographical name
    espn,1,abbreviation
    nfl,1,abbreviation
    shop,1,noun
    nation,2,noun
    nfc,1,abbreviation
    east,1,adverb
    south,1,adverb
    afc,1,abbreviation
    why,1,adverb
    cowboy,2,noun
    defense,2,noun
    jan,1,abbreviation
    pm,1,abbreviation
    dan,1,noun
    tweet,1,noun
    thursday,2,noun
    jerry,2,noun
    jones,1,noun
    showing,1,noun
    headed,2,adjective
    leadership,3,noun
    move,1,verb
    coherent,3,adjective
    vision,2,noun
    jean,1,noun
    taylor,2,biographical name
    agree,1,verb
    want,1,verb
    know,1,verb
    reasoning,1,noun
    behind,2,adverb or adjective
    defensive,3,adjective
    coordinate,4,adjective
    rob,1,verb
    ryan,2,biographical name
    monte,2,noun
    switch,1,noun
    alignment,2,noun
    discuss,2,transitive verb
    here,1,adverb
    financial,3,adjective
    motivate,3,transitive verb
    cap,1,noun
    problem,2,noun
    tie,1,noun
    player,2,noun
    anthony,3,biographical name
    spencer,2,noun
    jay,1,noun
    fine,1,noun
    explanation,4,noun
    likely,2,adjective
    admit,2,verb
    hope,1,verb
    angry,2,adjective
    sake,1,noun
    shift,1,verb
    philosophy,4,noun
    rid,1,transitive verb
    hire,1,noun
    threaten,2,verb
    lost,1,adjective
    couple,2,noun
    let,1,transitive verb
    happen,2,intransitive verb
    actually,4,adverb
    drop,1,noun
    knee,1,noun
    say,1,verb
    certainly,3,adverb
    threat,1,noun
    head,1,noun
    coach,1,noun
    dumb,1,adjective
    decision,3,noun
    could,1,verbal auxiliary
    ever,2,adverb
    scheme,1,noun
    don,1,transitive verb
    laugh,1,verb
    days,1,adverb
    cheer,1,noun
    unsuccessful,4,adjective
    reassurance,4,noun
    plan,1,noun
    working,2,noun
    slowly,2,adverb
    reason,2,noun
    customer,3,noun
    personally,4,adverb
    hunch,1,verb
    money,2,noun
    thing,1,noun
    ll,1,abbreviation
    losing,1,adjective
    him,1,pronoun
    cheap,1,noun
    solution,3,noun
    rusher,2,noun
    tyrone,2,geographical name
    developing,4,adjective
    job,1,noun
    identify,3,verb
    linebacker,3,noun
    lee,1,noun
    top,1,noun
    playmaker,3,noun
    accentuate,4,transitive verb
    potential,3,adjective
    qualify,3,verb
    forward,2,adjective
    thinking,1,noun
    driven,1,adjective
    honest,2,adjective
    assessment,3,noun
    personnel,3,noun
    circumstance,3,noun
    truth,1,noun
    speculate,3,verb
    random,2,noun
    going,2,noun
    send,1,verb
    spiral,2,adjective
    off,1,adverb
    direction,3,noun
    tag,1,noun
    jason,2,noun
    hamilton,3,biographical name
    colt,1,noun
    question,2,noun
    hover,2,intransitive verb
    clear,1,adjective
    arian,3,adjective
    candidate,3,noun
    indianapolis,6,geographical name
    offensive,3,adjective
    luck,1,noun
    maintain,2,transitive verb
    growth,1,noun
    curve,1,adjective
    arizona,4,geographical name
    cardinal,3,noun
    initial,2,adjective
    answer,2,noun
    morning,2,noun
    chase,1,noun
    pep,1,noun
    enlarge,2,verb
    sport,1,verb
    smooth,1,adjective
    endow,2,transitive verb
    technical,3,adjective
    director,3,noun
    offense,2,noun
    lure,1,noun
    hard,1,adjective
    resist,2,verb
    back,1,noun
    guy,1,noun
    league,1,noun
    quarterback,3,noun
    love,1,noun
    campus,2,noun
    weather,2,noun
    alto,2,noun
    anything,2,pronoun
    professional,4,adjective
    promotion,3,noun
    raise,1,verb
    too,1,adverb
    recently,3,adverb
    interview,3,noun
    jet,1,noun
    bio,1,noun
    automatic,4,adjective
    comfort,2,transitive verb
    giant,2,noun
    adjustment,3,noun
    departure,3,noun
    manager,3,noun
    congratulate,4,transitive verb
    opportunity,5,noun
    exceptional,4,adjective
    past,1,adjective
    keeping,1,noun
    record,2,verb
    accomplishment,4,noun
    attractive,3,adjective
    judgment,2,noun
    select,2,adjective
    ba,1,symbol
    remarkable,3,adjective
    filling,2,noun
    friend,1,noun
    selfless,2,adjective
    football,2,noun
    acumen,2,noun
    competitive,4,adjective
    surely,2,adverb
    presence,2,noun
    wish,1,verb
    entire,2,adjective
    family,3,noun
    nothing,2,pronoun
    embark,2,verb
    chuck,1,verb
    excite,2,transitive verb
    man,1,noun
    absence,2,noun
    truly,2,adverb
    forever,3,adverb
    debt,1,noun
    chapter,2,noun
    final,2,adjective
    falcon,2,noun
    pat,1,noun
    championship,4,noun
    raven,2,noun
    nugget,2,noun
    knowledge,2,noun
    sunday,2,noun
    san,1,noun
    atlanta,3,geographical name
    ap,1,abbreviation
    photo,2,noun
    perry,2,noun
    tony,1,adjective
    tight,1,adjective
    stay,1,noun
    grounded,2,adjective
    matchup,2,noun
    rush,1,noun
    yard,1,noun
    division,3,noun
    round,1,adjective
    mobile,2,adjective
    regular,3,adjective
    seattle,3,geographical name
    russell,2,biographical name
    wilson,2,biographical name
    average,3,noun
    high,1,adjective
    per,1,preposition
    carry,2,verb
    face,1,noun
    option,2,noun
    play,1,noun
    accord,2,verb
    stat,1,noun
    once,1,adverb
    reeve,1,noun
    quadruple,3,verb
    bypass,2,noun
    surgery,2,noun
    super,2,adjective
    bowl,1,noun
    appearance,3,noun
    franchise,2,noun
    rival,2,noun
    until,2,preposition
    meeting,1,noun
    comeback,2,noun
    kid,1,noun
    ball,1,noun
    rally,2,verb
    marked,1,adjective
    winning,1,noun
    drive,1,verb
    minute,2,noun
    fourth,1,noun
    quarter,2,noun
    rest,1,noun
    eight,1,noun
    multiple,3,adjective
    regulation,4,noun
    jaguar,2,noun
    receiver,3,noun
    harry,2,transitive verb
    douglas,2,biographical name
    reception,3,noun
    red,1,adjective
    hot,1,adjective
    factor,2,noun
    smith,1,noun
    receive,2,verb
    touchdown,2,noun
    carolina,4,geographical name
    panther,2,noun
    tampa,2,geographical name
    bay,1,adjective
    buccaneer,3,noun
    qb,1,abbreviation
    warrior,2,noun
    joe,1,noun
    nine,1,noun
    tom,1,noun
    brady,2,biographical name
    manning,2,biographical name
    montana,2,geographical name
    terry,2,noun
    young,1,adjective
    troy,1,adjective
    eli,1,noun
    overall,2,adverb
    ben,1,adverb
    ride,1,verb
    ray,1,noun
    retiring,1,adjective
    middle,2,adjective
    emotional,3,adjective
    rallying,1,noun
    tackle,2,noun
    jonathan,3,noun
    negate,2,transitive verb
    penalty,3,noun
    field,1,noun
    snap,1,verb
    iron,1,noun
    week,1,noun
    repair,2,intransitive verb
    triceps,2,noun
    right,1,adjective
    newman,2,biographical name
    catch,1,verb
    crack,1,verb
    woeful,2,adjective
    flying,1,adjective
    trying,1,adjective
    exploit,2,noun
    rank,1,adjective
    downfield,2,adverb or adjective
    night,1,noun
    baltimore,3,biographical name
    younger,2,noun
    brother,2,noun
    motorcycle,4,noun
    accident,3,noun
    finished,2,adjective
    six,1,noun
    explosive,3,adjective
    denver,2,geographical name
    score,1,noun
    seventh,2,noun
    target,2,noun
    pressure,2,noun
    sack,1,noun
    career,2,noun
    away,1,adverb
    willy,2,noun
    lead,1,verb
    struggle,2,intransitive verb
    mar,1,transitive verb
    injury,3,noun
    achilles,2,noun
    tendon,2,noun
    biceps,2,noun
    total,2,adjective
    lately,2,adverb
    registered,1,adjective
    consecutive,4,adjective
    streak,1,noun
    least,1,adjective
    moving,1,adjective
    fisher,2,noun
    bill,1,noun
    fox,1,noun
    raider,2,noun
    flores,2,biographical name
    advance,2,verb
    lose,1,verb
    reid,1,biographical name
    madden,2,verb
    knox,1,biographical name
    england,2,geographical name
    georgia,2,geographical name
    dome,1,noun
    equalizer,3,noun
    facing,2,noun
    victory,3,noun
    matt,1,abbreviation
    defeat,2,transitive verb
    aaron,2,noun
    rodgers,2,biographical name
    center,2,noun
    josh,1,verb
    tough,1,adjective
    task,1,noun
    heading,2,noun
    venue,2,noun
    month,1,noun
    loss,1,noun
    enter,2,verb
    minimum,3,noun
    gone,1,adjective
    mix,1,verb
    ninth,1,noun
    pittsburgh,2,geographical name
    figure,2,noun
    grow,1,verb
    philadelphia,4,geographical name
    green,1,adjective
    side,1,noun
    lone,1,adjective
    flourish,2,verb
    wrapping,1,noun
    fame,1,noun
    counterpart,3,noun
    vernon,2,biographical name
    numbers,2,noun plural but singular in construction
    impact,2,verb
    opponent,3,noun
    low,1,intransitive verb
    completion,3,noun
    percentage,3,noun
    nice,1,adjective
    oppose,2,transitive verb
    ranking,2,adjective
    worse,1,adjective
    buffalo,3,noun
    rushing,2,noun
    pistol,2,noun
    pct,1,abbreviation
    respectable,3,adjective
    handed,2,adjective
    cam,1,noun
    newton,2,noun
    account,2,noun
    nearly,2,adverb
    yardage,2,noun
    covering,3,noun
    formation,3,noun
    starter,2,noun
    attention,3,noun
    fitting,1,adjective
    respect,2,noun
    tenured,2,adjective
    frank,1,adjective
    gore,1,noun
    comfortable,3,adjective
    item,1,adverb
    james,1,noun
    walker,2,noun
    gillette,2,biographical name
    stadium,3,noun
    generation,4,noun
    square,1,noun
    retire,2,verb
    fact,1,noun
    decided,3,adjective
    sixth,1,noun
    arguably,3,adverb
    defend,2,verb
    unhappy,3,adjective
    kick,1,verb
    coverage,3,noun
    kickoff,2,noun
    houston,2,biographical name
    bad,1,adjective
    bronco,2,noun
    punt,1,noun
    clean,1,adjective
    difference,3,noun
    pro,1,noun
    easy,1,adjective
    brandon,2,geographical name
    step,1,noun
    lineup,2,noun
    solid,2,adjective
    dual,2,adjective
    increase,2,verb
    tempo,2,noun
    push,1,verb
    overtime,2,noun
    straight,1,adjective
    advantage,3,noun
    bye,1,noun
    reasonable,3,adjective
    healthy,1,adjective
    exception,3,noun
    frustrate,2,transitive verb
    orchestrate,3,transitive verb
    tremendous,3,adjective
    replacement,3,noun
    doing,2,noun
    satisfy,3,verb
    reportedly,4,adverb
    todd,1,biographical name
    minded,2,adjective
    deserved,2,adjective
    seek,1,verb
    priority,4,noun
    prevent,2,verb
    pursue,2,verb
    lateral,3,adjective
    valuable,2,adjective
    substance,2,noun
    bit,1,noun
    loose,1,adjective
    cannon,2,noun
    publicly,3,adverb
    everyone,3,pronoun
    impression,3,noun
    outfox,2,transitive verb
    flippant,2,adjective
    fare,1,intransitive verb
    poorly,2,adverb
    someone,2,pronoun
    seriously,4,adverb
    eagle,2,noun
    bowles,1,biographical name
    case,1,noun
    philip,2,biographical name
    rivers,2,biographical name
    williamson,3,biographical name
    reclamation,4,noun
    continued,1,adjective
    arsenal,3,noun
    friday,2,noun
    reich,1,biographical name
    former,2,adjective
    longtime,2,adjective
    backup,2,noun
    serving,1,noun
    charger,2,noun
    mccoy,2,noun
    obvious,3,adjective
    background,2,noun
    fall,1,verb
    turnover,2,noun
    believe,2,verb
    paramount,3,adjective
    department,3,noun
    brown,1,adjective
    authority,4,noun
    cleveland,2,biographical name
    jimmy,2,noun
    talk,1,verb
    chief,1,adjective
    executive,4,adjective
    officer,3,noun
    banner,2,noun
    rounded,2,adjective
    introductory,5,adjective
    consensus,3,noun
    really,3,adverb
    rumor,2,noun
    negative,3,adjective
    backlash,2,noun
    media,2,noun
    touched,1,adjective
    upon,2,preposition
    reaction,3,noun
    reporter,3,noun
    walk,1,verb
    room,1,noun
    positive,3,adjective
    shy,1,adjective
    challenge,2,verb
    spoken,2,adjective
    spent,1,adjective
    lot,1,noun
    excellent,3,adjective
    talent,2,noun
    near,1,adverb
    analyst,3,noun
    network,2,noun
    pick,1,verb
    gordon,2,biographical name
    charge,1,noun
    joining,2,noun
    put,1,verb
    oh,1,interjection
    trouble,2,verb
    different,3,adjective
    opposed,2,adjective
    personality,5,noun
    watch,1,verb
    perspective,3,noun
    inside,2,noun
    house,1,noun
    farmer,2,noun
    minnesota,4,geographical name
    paton,2,biographical name
    deciding,3,adjective
    finding,1,noun
    person,2,noun
    build,1,verb
    assemble,3,verb
    compliment,3,noun
    attitude,3,noun
    ethic,2,noun
    particular,4,adjective
    program,2,noun
    necessarily,5,adverb
    charade,2,noun
    gm,1,abbreviation
    bradley,2,biographical name
    caldwell,2,biographical name
    duo,1,noun
    energy,3,noun
    passion,2,noun
    tendency,3,noun
    largely,2,adverb
    opposite,3,adjective
    departed,3,adjective
    gene,1,noun
    phil,1,abbreviation
    sears,1,biographical name
    feel,1,verb
    coming,2,noun
    excitement,3,noun
    hold,1,verb
    shad,1,noun
    khan,1,noun
    chance,1,noun
    visit,2,verb
    accomplish,3,transitive verb
    our,1,adjective
    mesh,1,noun
    exciting,3,adjective
    alone,1,adjective
    cure,1,noun
    ail,1,verb
    jacksonville,3,geographical name
    appreciate,4,verb
    thank,1,transitive verb
    humble,2,adjective
    certificate,4,noun
    catholic,3,adjective
    desire,2,verb
    parent,2,noun
    nickname,2,noun
    cross,1,noun
    path,1,noun
    alma,2,geographical name
    mater,2,noun
    dakota,3,noun
    state,1,noun
    scouting,2,noun
    colleague,2,noun
    staff,1,noun
    business,2,noun
    regarding,3,preposition
    strength,1,noun
    hurt,1,verb
    cause,1,noun
    grouping,2,noun
    diversity,4,noun
    spread,1,verb
    zone,1,noun
    difficulty,4,noun
    something,2,pronoun
    jack,1,noun
    del,1,abbreviation
    rio,1,abbreviation
    unable,2,adjective
    able,1,adjective
    effectively,4,adverb
    function,2,noun
    timetable,3,noun
    contender,3,noun
    phone,1,noun
    decline,2,verb
    flynn,1,biographical name
    contract,2,noun
    blaine,1,biographical name
    chad,1,noun
    genuine,3,adjective
    obviously,4,adverb
    bizarre,2,adjective
    story,2,noun
    dame,1,noun
    fake,1,transitive verb
    girlfriend,2,noun
    still,1,adjective
    whether,2,pronoun
    victim,2,noun
    elaborate,3,adjective
    prank,1,noun
    affect,2,noun
    standing,2,adjective
    cal,1,abbreviation
    neat,1,noun
    baggage,2,noun
    overlook,2,transitive verb
    prior,2,noun
    revelations,4,noun plural but singular in construction
    jr,1,abbreviation
    mock,1,verb
    projection,3,noun
    stuff,1,noun
    decent,2,adjective
    blackburn,2,biographical name
    leader,2,noun
    instinct,2,noun
    ability,3,noun
    spark,1,noun
    address,2,verb
    vote,1,noun
    bring,1,verb
    circus,2,noun
    disdain,2,noun
    invite,2,transitive verb
    fresh,1,adjective
    locker,2,noun
    bet,1,noun
    decide,2,verb
    profile,2,noun
    needs,1,adverb
    plus,1,adjective
    speed,1,noun
    athletic,3,adjective
    elsewhere,2,adverb
    especially,4,adverb
    washington,3,biographical name
    redskin,2,noun
    fletcher,2,noun
    soon,1,adverb
    down,1,adverb
    imagine,2,verb
    lie,1,intransitive verb
    fired,1,adjective
    ago,1,adjective or adverb
    puzzling,1,adjective
    announcement,3,noun
    rebuild,2,verb
    roster,2,noun
    foundation,3,noun
    trent,1,geographical name
    richardson,3,biographical name
    ward,1,noun
    full,1,adjective
    choice,1,noun
    regime,2,noun
    blasted,2,adjective
    hear,1,verb
    serve,1,verb
    skeptical,3,adjective
    calling,2,noun
    panic,2,adjective
    disaster,3,noun
    supplemental,4,adjective
    wasted,1,adjective
    prove,1,verb
    surprising,1,adjective
    addition,3,noun
    far,1,adverb
    refuse,2,verb
    reality,4,noun
    afford,2,transitive verb
    constant,2,adjective
    spew,1,verb
    whatever,3,pronoun
    company,3,noun
    white,1,adjective
    afraid,1,adjective
    mind,1,noun
    half,1,noun
    heck,1,noun
    execute,3,verb
    complacent,3,adjective
    relaxed,2,adjective
    lull,1,transitive verb
    credit,2,noun
    sure,1,adjective
    disastrous,3,adjective
    bottom,2,noun
    foot,1,noun
    accelerator,5,noun
    big,1,adjective
    admire,2,verb
    underrate,3,transitive verb
    definite,3,adjective
    seat,1,noun
    table,2,noun
    successful,3,adjective
    verge,1,noun
    reach,1,verb
    weathering,1,noun
    storm,1,noun
    kelly,2,biographical name
    howard,2,biographical name
    chip,1,noun
    stamp,1,verb
    organization,5,noun
    pushing,2,adjective
    ultimately,4,adverb
    seal,1,noun
    triumph,2,noun
    trust,1,noun
    large,1,adjective
    significantly,5,adverb
    rich,1,adjective
    hofmann,2,biographical name
    pretty,2,adjective
    turnaround,2,noun
    public,2,adjective
    emerge,1,intransitive verb
    process,2,noun
    powerful,3,adjective
    wince,1,intransitive verb
    whenever,3,conjunction
    circle,2,noun
    control,2,verb
    nuance,2,noun
    willing,2,adjective
    empire,2,noun
    building,2,noun
    everybody,3,pronoun
    structure,2,noun
    operation,4,noun
    few,1,pronoun, plural in construction
    collapse,2,verb
    continual,4,adjective
    peg,1,noun
    hole,1,noun
    arrangement,3,noun
    success,2,noun
    operate,3,verb
    develop,3,verb
    thirst,1,noun
    greater,2,adjective
    acquire,2,transitive verb
    inability,4,noun
    knowing,1,noun
    jeffrey,2,biographical name
    confusion,3,noun
    necessitate,4,transitive verb
    intertwine,3,verb
    themselves,2,pronoun plural
    occupy,3,transitive verb
    easily,3,adverb
    empty,2,adjective
    retrospect,3,noun
    page,1,noun
    headline,2,noun
    scoreboard,2,noun
    preview,2,transitive verb
    ticket,2,noun
    cbs,1,abbreviation
    subscribe,2,verb
    twitter,2,verb
    wrap,1,verb
    dolphin,2,noun
    lion,2,noun
    ram,1,noun
    agency,2,noun
    acc,1,abbreviation
    schedule,2,noun
    analysis,3,noun
    alabama,3,geographical name
    crimson,2,noun
    tide,1,noun
    bart,1,abbreviation
    scott,1,biographical name
    ten,1,noun
    powell,2,biographical name
    college,2,noun
    calvin,2,biographical name
    pace,1,noun
    cincinnati,3,geographical name
    tiger,2,noun
    connecticut,4,geographical name
    husky,1,adjective
    harris,2,biographical name
    derrick,2,noun
    mason,2,noun
    detroit,2,geographical name
    duke,1,noun
    blue,1,adjective
    devil,2,noun
    florida,3,geographical name
    gator,2,noun
    seminole,3,noun
    bulldog,2,noun
    tech,1,noun
    yellow,2,adjective
    jacket,2,noun
    nick,1,noun
    iowa,2,geographical name
    hawkeye,2,noun
    wc,1,abbreviation
    namath,2,biographical name
    jordan,2,biographical name
    kansas,2,geographical name
    louisville,3,geographical name
    maryland,2,geographical name
    terrapin,3,noun
    miami,2,noun
    hurricane,3,noun
    wright,1,noun
    michigan,3,geographical name
    spartan,2,noun
    wolverine,3,noun
    golden,2,adjective
    viking,2,noun
    mississippi,4,geographical name
    rebel,2,adjective
    missouri,3,geographical name
    muhammad,3,biographical name
    nebraska,3,geographical name
    cornhusker,3,noun
    folk,1,noun
    tar,1,noun
    heel,1,noun
    northwestern,3,adjective
    wildcat,2,noun
    oakland,2,geographical name
    ohio,1,geographical name
    buckeye,2,noun
    oklahoma,3,geographical name
    pac,1,abbreviation
    penn,1,abbreviation
    nittany,3,geographical name
    boilermaker,4,noun
    rabid,2,adjective
    rapid,2,adjective
    rex,1,noun
    scarlet,2,noun
    knight,1,noun
    sec,1,adjective
    holmes,1,biographical name
    greene,1,biographical name
    bull,1,noun
    stephen,2,biographical name
    hill,1,noun
    syracuse,3,geographical name
    orange,2,noun
    tennessee,3,geographical name
    titan,2,noun
    volunteer,3,noun
    tim,1,abbreviation
    cruz,1,biographical name
    virginia,3,geographical name
    cavalier,3,noun
    wake,1,verb
    forest,2,noun
    demon,2,noun
    wayne,1,biographical name
    hunter,2,noun
    mountaineer,3,noun
    wisconsin,3,geographical name
    badger,2,noun
    hour,1,noun
    video,2,noun
    marc,1,noun
    hit,1,verb
    poll,1,noun
    arrest,2,transitive verb
    airport,2,noun
    update,2,transitive verb
    mlb,1,abbreviation
    nba,1,abbreviation
    nhl,1,abbreviation
    nascar,1,abbreviation
    soccer,2,noun
    tennis,2,noun
    boxing,2,noun
    mma,1,abbreviation
    insider,3,noun
    sn,1,Symbol
    radio,2,adjective
    playbook,2,noun
    fantasy,3,noun
    ne,1,symbol
    odds,1,noun plural but singular or plural in construction
    daily,2,adjective
    forum,2,noun
    attendance,3,noun
    transaction,3,noun
    mn,1,symbol
    hq,1,abbreviation
    advertise,3,verb
    sales,1,adjective
    ad,1,noun
    correction,3,noun
    patent,2,adjective
    supply,2,verb
    internet,3,noun
    venture,2,verb
    privacy,3,noun
    policy,3,noun
    safety,2,noun
    california,4,geographical name
    applicable,4,adjective
    reserved,2,adjective
    buster,2,noun
    indian,3,noun
    brave,1,adjective
    splash,1,verb
    dodger,2,noun
    baseball,2,noun
    tonight,2,adverb
    sweep,1,verb
    leaderboard,3,noun
    overtake,2,transitive verb
    phillips,2,adjective
    gem,1,noun
    tally,2,noun
    royal,2,adjective
    race,1,noun
    honor,2,noun
    cameron,3,biographical name
    dive,1,verb
    snide,1,adjective
    climb,1,verb
    wall,1,noun
    steal,1,verb
    twin,1,noun
    host,1,noun
    curt,1,adjective
    schilling,2,noun
    pedro,2,biographical name
    jon,1,abbreviation
    boone,1,biographical name
    gehrig,2,biographical name
    orel,1,geographical name
    barry,2,biographical name
    larkin,2,biographical name
    stark,1,adjective
    singleton,3,noun
    rick,1,noun
    al,1,symbol
    oriole,2,noun
    toronto,3,geographical name
    angel,2,noun
    athletics,3,noun plural but singular or plural in construction
    mariner,3,noun
    ranger,2,noun
    nl,1,abbreviation
    marlin,2,noun
    met,1,noun,
    cub,1,noun
    milwaukee,3,geographical name
    pirate,2,noun
    colorado,4,geographical name
    wbc,1,abbreviation
    check,1,noun
    blues,1,noun plural but singular or plural in construction
    sign,1,noun
    wade,1,verb
    redden,2,verb
    official,3,noun
    principal,3,adjective
    defenseman,3,noun
    armstrong,2,biographical name
    complement,3,noun
    stability,4,noun
    core,1,noun
    hockey,2,noun
    whale,1,noun
    goal,1,noun
    assist,2,verb
    veteran,3,noun
    ottawa,3,noun
    saskatchewan,4,geographical name
    native,2,adjective
    originally,4,adverb
    islander,3,noun
    arrive,2,intransitive verb
    weekend,2,noun
    undergo,3,transitive verb
    physical,3,adjective
    forecast,2,verb
    intense,2,adjective
    competition,4,noun
    hallmark,2,noun
    nightly,2,adjective
    invariably,3,adverb
    ups,1,abbreviation
    defy,2,transitive verb
    prognostication,5,noun
    sprint,1,intransitive verb
    frenetic,3,adjective
    saying,1,noun
    considering,4,preposition
    stanley,2,biographical name
    repeated,3,adjective
    champion,3,noun
    qualified,3,adjective
    eastern,2,adjective
    trophy,2,noun
    eventual,3,adjective
    kings,1,noun plural but singular in construction
    clinch,1,verb
    berth,1,noun
    apr,1,abbreviation
    nashville,2,geographical name
    predator,3,noun
    seed,1,noun
    grab,1,verb
    determined,3,adjective
    reduce,2,verb
    eighth,1,noun
    edmonton,3,geographical name
    calgary,3,geographical name
    becoming,3,adjective
    saturday,3,noun
    nbc,1,abbreviation
    cbc,1,abbreviation
    canada,3,geographical name
    winnipeg,3,geographical name
    montreal,3,geographical name
    anaheim,3,noun
    vancouver,3,biographical name
    ny,1,abbreviation
    phoenix,2,noun
    tuesday,2,noun
    wednesday,2,noun
    rivalry,3,noun
    feb,1,abbreviation
    commencement,3,noun
    whitney,2,biographical name
    coyote,2,noun
    peterborough,4,geographical name
    ontario,3,geographical name
    across,1,adverb
    wear,1,verb
    favorite,3,noun
    jersey,2,noun
    celebrate,3,verb
    hero,2,noun
    doubleheader,4,noun
    evening,2,noun
    noon,1,noun
    vs,1,abbreviation
    soldier,2,noun
    deadline,2,noun
    ncaa,1,abbreviation
    frozen,2,adjective
    consol,1,abbreviation
    monday,2,noun
    drawing,2,noun
    memorial,4,adjective
    saskatoon,3,noun
    chat,1,verb
    sarah,2,noun
    goldstein,2,biographical name
    prepare,2,verb
    shorten,2,verb
    wonder,2,noun
    compete,2,intransitive verb
    asterisk,3,noun
    contention,3,noun
    rise,1,intransitive verb
    pacific,3,adjective
    instead,2,adverb
    effect,2,noun
    outcome,2,noun
    purpose,2,noun
    tell,1,verb
    bruin,2,noun
    senator,3,noun
    capital,3,noun
    maple,2,noun
    leaf,1,noun
    lightning,2,noun
    saber,2,noun
    wing,1,noun
    canuck,2,noun
    shark,1,noun
    avalanche,3,noun
    flame,1,noun
    duck,1,noun
    oiler,2,noun
    columbus,3,biographical name
    courtesy,3,noun
    elias,2,noun
    bureau,2,noun
    roundup,2,noun
    subtraction,3,noun
    sheldon,2,biographical name
    bryan,2,biographical name
    allen,2,biographical name
    lw,1,abbreviation
    daniel,2,noun
    rw,1,abbreviation
    blake,1,biographical name
    johnson,2,noun
    adam,2,noun
    pardie,2,interjection
    roy,1,geographical name
    brad,1,noun
    alexander,4,noun
    sutter,2,biographical name
    brunet,2,noun
    peter,2,intransitive verb
    adrian,2,biographical name
    korsakoff,3,adj
    nash,1,biographical name
    jonas,2,noun
    stuart,2,adjective
    nail,1,noun
    barker,2,noun
    garrison,3,noun
    none,1,pronoun, singular or plural in construction
    zach,1,abbreviation
    mitchell,2,biographical name
    guillaume,2,biographical name
    colby,2,noun
    anders,2,biographical name
    czech,1,noun
    republic,3,noun
    luke,1,noun
    carl,1,noun
    cerenkov,3,adj
    sullivan,3,biographical name
    dominic,3,biographical name
    moore,1,biographical name
    sami,2,noun
    bruno,2,biographical name
    lockout,2,noun
    challenged,1,adjective
    milestone,2,noun
    dal,1,noun
    ana,1,adverb
    patrick,2,biographical name
    sj,1,abbreviation
    edm,1,abbreviation
    tb,1,abbreviation
    thornton,2,biographical name
    agent,1,noun
    marian,3,adjective
    chi,1,noun
    goaltending,3,noun
    pit,1,noun
    fla,1,abbreviation
    shutout,2,noun
    nabokov,3,biographical name
    youth,1,noun
    shine,1,verb
    pierre,1,geographical name
    lebrun,2,biographical name
    matthew,2,noun
    waiver,2,noun
    signed,1,adjective
    waive,1,transitive verb
    randy,1,adjective
    carlyle,2,biographical name
    tool,1,noun
    disposal,3,noun
    wrong,1,noun
    shape,1,verb
    viable,3,adjective
    asset,2,noun
    transparency,4,noun
    accountability,5,noun
    earn,1,transitive verb
    camp,1,noun
    show,1,verb
    certain,2,adjective
    luxury,3,noun
    bar,1,noun
    burnside,2,biographical name
    doubt,1,verb
    might,1,verbal auxiliary
    heretical,4,adjective
    stature,2,noun
    swap,1,verb
    cane,1,noun
    dropping,1,noun
    plate,1,noun
    cooke,1,biographical name
    tyler,2,biographical name
    kennedy,3,biographical name
    trio,1,noun
    extremely,3,adverb
    effective,3,adjective
    rely,2,intransitive verb
    malkin,2,noun
    sidney,2,biographical name
    crosby,2,biographical name
    himself,2,pronoun
    killing,2,noun
    draw,1,verb
    gee,1,verb
    comparison,4,noun
    bright,1,adjective
    dynamic,3,adjective
    confront,2,transitive verb
    benefit,3,noun
    standout,2,noun
    opener,2,noun
    afternoon,3,noun
    myself,2,pronoun
    wise,1,noun
    taste,1,verb
    hopefully,3,adverb
    establish,3,transitive verb
    marquee,2,noun
    interesting,4,adjective
    observation,4,noun
    accompany,4,verb
    million,2,noun
    extension,3,noun
    bonus,2,noun
    clause,1,noun
    course,1,noun
    loaded,2,adjective
    salary,3,noun
    variance,3,noun
    hop,1,verb
    notice,2,noun
    amount,1,intransitive verb
    guess,1,verb
    dieldrin,2,noun
    figured,1,adjective
    protect,2,verb
    escrow,2,noun
    payment,2,noun
    allowance,3,noun
    real,2,adjective
    revenue,3,noun
    speak,1,verb
    belief,2,noun
    importantly,4,adverb
    captain,2,noun
    superstar,3,noun
    winger,2,noun
    tackling,2,noun
    maximum,3,noun
    act,1,noun
    magnet,2,noun
    respective,3,adjective
    board,1,noun
    slate,1,noun
    restricted,3,adjective
    apple,2,noun
    office,2,noun
    slice,1,verb
    intriguing,3,adjective
    grizzled,2,adjective
    fledge,1,verb
    rookie,2,noun
    wife,1,noun
    pregnant,2,adjective
    father,2,noun
    baby,2,noun
    sting,1,verb
    teammate,2,noun
    throwback,2,noun
    quits,1,adjective
    failing,2,noun
    shock,1,noun
    settle,2,verb
    labor,2,noun
    pack,1,noun
    sleep,1,noun
    nights,1,adverb
    pull,1,verb
    trigger,2,noun
    mood,1,noun
    gnaw,1,verb
    telling,1,adjective
    hang,1,verb
    oct,1,abbreviation
    cost,1,noun
    limited,3,adjective
    tank,1,noun
    eliminate,3,verb
    realize,3,transitive verb
    belt,1,noun
    ice,1,noun
    fond,1,adjective
    picture,2,noun
    southern,2,adjective
    cherish,2,transitive verb
    enjoy,2,verb
    living,1,adjective
    heart,1,noun
    perhaps,2,adverb
    intrigue,2,noun
    tv,1,noun
    bolt,1,noun
    expectation,4,noun
    halfway,2,adjective
    prelude,2,noun
    acquisition,4,noun
    hand,1,noun
    calm,1,noun
    whoa,1,verb imperative
    cart,1,noun
    horse,1,noun
    fair,1,adjective
    undue,2,adjective
    quiet,2,noun
    goalie,2,noun
    prayer,1,noun
    gamble,2,verb
    learned,1,adjective
    worthy,2,adjective
    skill,1,intransitive verb
    tune,1,noun
    roll,1,noun
    dice,1,noun
    somebody,2,pronoun
    absolutely,4,adverb
    turn,1,verb
    stud,1,noun
    hey,1,interjection
    quirk,1,noun
    elite,1,noun
    goaltender,3,noun
    patch,1,noun
    price,1,noun
    along,1,preposition
    everything,3,pronoun
    upside,2,noun
    bargain,2,noun
    payroll,2,noun
    perfect,2,adjective
    olympic,2,adjective
    gold,1,noun
    market,2,noun
    martin,2,noun
    shadow,2,noun
    adjust,2,verb
    fortunate,3,adjective
    giardia,2,noun
    number,2,noun
    ease,1,noun
    burden,2,noun
    gradual,3,noun
    pickup,2,noun
    deliver,3,verb
    demeanor,3,noun
    arena,2,noun
    skating,1,noun
    onto,2,preposition
    slow,1,adjective
    earth,1,noun
    nobody,2,pronoun
    legendary,3,adjective
    incredible,4,adjective
    holland,2,noun
    norris,2,biographical name
    bobby,2,noun
    orr,1,biographical name
    committee,3,noun
    realism,3,noun
    upbeat,2,noun
    defender,3,noun
    bruising,1,adjective
    dependable,3,adjective
    force,1,noun
    ericsson,3,biographical name
    killer,2,noun
    eat,1,verb
    legitimate,4,adjective
    plenty,2,noun
    deny,2,transitive verb
    tandem,2,noun
    therefore,2,adverb
    tinker,2,noun
    mull,1,verb
    merit,2,noun
    ahead,1,adverb or adjective
    jackhammer,3,noun
    huge,1,adjective
    test,1,noun
    forecheck,2,intransitive verb
    pout,1,verb
    cry,1,verb
    spill,1,verb
    milk,1,noun
    obstacle,3,noun
    ultimate,3,adjective
    decade,2,noun
    forge,1,noun
    surprise,2,noun
    minor,2,adjective
    sudden,2,adjective
    butler,2,noun
    jacob,2,noun
    prospect,2,noun
    forefront,2,noun
    masterful,3,adjective
    bench,1,noun
    route,1,noun
    fun,1,noun
    aggressive,3,adjective
    integrity,4,noun
    responsible,4,adjective
    somewhere,2,adverb
    shotgun,2,noun
    bergen,2,geographical name
    eventually,4,adverb
    surplus,2,noun
    sit,1,verb
    hawks,1,biographical name
    weakness,2,noun
    damien,2,biographical name
    swiss,1,noun
    hype,1,noun
    surround,2,transitive verb
    arrival,3,noun
    ocean,1,noun
    inescapable,4,adjective
    skeptic,2,noun
    finish,2,verb
    unless,2,conjunction
    scout,1,verb
    fight,1,verb
    heavy,1,adjective
    traffic,2,noun
    small,1,adjective
    rink,1,noun
    skater,2,noun
    lake,1,noun
    british,2,noun
    overseas,2,adverb
    switzerland,3,geographical name
    bid,1,verb
    client,2,noun
    silver,2,noun
    lining,2,noun
    zug,1,geographical name
    stoppage,2,noun
    chemistry,3,noun
    space,1,noun
    ways,1,noun plural but singular in construction
    eligible,4,adjective
    calder,2,biographical name
    sept,1,noun
    rested,2,adjective
    ready,1,adjective
    otherwise,3,pronoun
    funny,2,adjective
    anyway,2,adverb
    bowman,2,noun
    forwards,2,adverb
    health,1,noun
    concussion,3,noun
    deck,1,noun
    brutal,2,adjective
    recover,3,verb
    extended,3,adjective
    heal,1,verb
    feeling,2,noun
    totally,3,adverb
    optimism,3,noun
    corey,2,biographical name
    riding,1,noun
    rebound,2,verb
    sensational,4,adjective
    helping,2,noun
    upset,2,verb
    magnificent,4,adjective
    tale,1,noun
    stretch,1,verb
    fluke,1,noun
    mentality,4,noun
    badly,2,adverb
    steady,1,adjective
    bounce,1,verb
    campaign,2,noun
    slump,1,intransitive verb
    albeit,3,conjunction
    maturity,4,noun
    sacco,2,biographical name
    diminished,3,adjective
    response,2,noun
    refocus,3,verb
    commit,2,verb
    capable,3,adjective
    gabriel,3,noun
    newly,2,adverb
    intact,2,adjective
    milan,2,geographical name
    typical,3,adjective
    downs,1,geographical name
    energetic,4,adjective
    soul,1,noun
    representative,5,adjective
    newport,2,geographical name
    negotiate,4,verb
    above,1,adverb
    sens,1,geographical name
    deplete,2,transitive verb
    cowen,2,biographical name
    finger,2,noun
    thin,1,adjective
    murray,2,biographical name
    depth,1,noun
    wait,1,verb
    rumbling,1,noun
    insist,2,verb
    patient,2,adjective
    regardless,3,adjective
    anybody,2,pronoun
    town,1,noun
    terrible,3,adjective
    netminder,3,noun
    utmost,2,adjective
    destination,4,noun
    shed,1,verb
    light,1,noun
    indeed,2,adverb
    brian,2,biographical name
    burke,1,transitive verb
    firing,2,noun
    despite,2,noun
    clearly,2,adverb
    underline,3,transitive verb
    body,1,noun
    gates,1,biographical name
    risk,1,noun
    ramp,1,verb
    poor,1,adjective
    whichever,3,adjective
    await,1,verb
    buyout,2,noun
    union,1,noun
    ply,1,verb
    pay,1,verb
    limbo,2,noun
    compliance,3,noun
    jeopardize,3,transitive verb
    saving,1,noun
    injure,2,transitive verb
    argue,2,verb
    sort,1,noun
    agreement,2,noun
    shall,1,verb
    solve,1,verb
    blood,1,noun
    thrasher,2,noun
    cover,2,verb
    windsor,1,biographical name
    sun,1,noun
    author,2,noun
    true,1,adjective
    crime,1,noun
    deadly,2,adjective
    innocence,3,noun
    spend,1,verb
    canadian,4,noun
    columnist,3,noun
    panelist,3,noun
    constitution,4,noun
    sporting,2,adjective
    choose,1,verb
    filter,2,noun
    rt,1,abbreviation
    chef,1,noun
    mask,1,noun
    dec,1,abbreviation
    recovery,3,noun
    aggravating,1,adjective
    skate,1,noun
    kris,1,noun
    questionable,3,adjective
    testing,2,adjective
    remember,3,verb
    ufa,1,geographical name
    leave,1,verb
    hitchcock,2,biographical name
    heat,1,verb
    la,1,interjection
    suspend,2,verb
    lv,1,abbreviation
    worth,1,intransitive verb
    disappointed,4,adjective
    mighty,1,adjective
    dean,1,noun
    carey,2,biographical name
    tomorrow,3,adverb
    maintenance,3,noun
    isle,1,noun
    cache,1,noun
    web,1,noun
    snafu,2,noun
    atlantic,3,adjective
    northeast,2,adverb
    northwest,2,adverb
    glance,1,verb
    relative,3,noun
    award,1,transitive verb
    register,3,noun
    id,1,noun
    google,2,transitive verb
    yahoo,2,noun
    mail,1,noun
    please,1,verb
    valid,2,adjective
    password,2,noun
    forget,2,verb
    store,1,transitive verb
    woman,2,noun
    training,2,noun
    europa,3,noun
    euro,2,noun
    amateur,3,noun
    region,2,noun
    ssi,1,abbreviation
    err,1,intransitive verb
    arrow,2,noun
    examination,5,noun
    illustrious,4,adjective
    ajax,1,noun
    resumption,3,noun
    manchester,3,geographical name
    fc,1,abbreviation
    paris,2,noun
    madrid,2,geographical name
    cf,1,abbreviation
    exacting,3,adjective
    italy,3,geographical name
    spain,1,geographical name
    rennes,1,geographical name
    rejoice,2,verb
    inter,2,transitive verb
    scratch,1,verb
    itch,1,verb
    lambert,2,noun
    unveiled,2,adjective
    xi,1,noun
    digest,2,noun
    fa,1,noun
    reveal,2,transitive verb
    action,2,noun
    mpa,1,abbreviation
    sk,1,abbreviation
    hannover,3,geographical name
    lyonnais,3,geographical name
    entertain,3,verb
    gaillard,2,biographical name
    lyon,2,biographical name
    bavaria,3,geographical name
    legal,2,adjective
    fixing,2,noun
    cancer,2,noun
    secretary,3,noun
    determination,5,noun
    corruption,3,noun
    referee,3,noun
    guest,1,noun
    highlight,2,noun
    additional,4,adjective
    assistant,3,noun
    activity,4,noun
    budapest,3,geographical name
    packed,1,adjective
    agenda,2,noun
    barcelona,4,geographical name
    factory,3,noun
    loading,2,noun
    boost,1,verb
    ashley,2,geographical name
    celtic,2,adjective
    youngster,2,noun
    height,1,noun
    grassroots,2,adjective
    rossi,2,biographical name
    midfield,2,noun
    walcott,2,biographical name
    pen,1,transitive verb
    oust,1,transitive verb
    loan,1,noun
    extend,2,verb
    appointment,3,noun
    eye,1,noun
    secure,2,adjective
    crown,1,noun
    spur,1,noun
    tottenham,3,geographical name
    midfielder,3,noun
    sustain,2,transitive verb
    premier,2,adjective
    psg,1,abbreviation
    southampton,3,geographical name
    wales,1,geographical name
    relish,2,noun
    user,2,noun
    gerard,2,biographical name
    sight,1,noun
    ac,1,abbreviation
    copy,1,noun
    ambition,3,noun
    glory,2,noun
    heartache,2,noun
    ramos,2,biographical name
    eindhoven,3,geographical name
    calendar,3,noun
    service,2,noun
    library,2,noun
    accredit,3,transitive verb
    blocked,1,adjective
    broadcast,2,adjective
    anti,2,noun
    doping,1,noun
    medical,3,adjective
    foursquare,2,adjective
    feed,1,verb
    feedback,2,noun
    faq,1,noun
    logo,1,noun
    copyright,2,noun
    rakudo,3,noun

`baseballtonight.csv`:

    usa,1,abbreviation
    more,1,adjective
    asia,1,geographical name
    australia,3,geographical name
    brazil,2,geographical name
    united,1,adjective
    kingdom,2,noun
    boston,2,noun
    chicago,3,geographical name
    dallas,2,biographical name
    los,1,abbreviation
    new,1,adjective
    york,1,biographical name
    espn,1,abbreviation
    mlb,1,abbreviation
    shop,1,noun
    winter,2,noun
    and,1,conjunction
    tim,1,abbreviation
    on,1,preposition
    the,1,definite article
    michael,2,noun
    young,1,adjective
    what,1,pronoun
    have,1,verb
    best,1,adjective
    chance,1,noun
    to,1,preposition
    land,1,noun
    josh,1,verb
    hamilton,3,biographical name
    make,1,verb
    a,1,noun
    for,1,preposition
    fresh,1,adjective
    start,1,verb
    minute,2,noun
    down,1,adverb
    with,1,preposition
    late,1,adjective
    charge,1,noun
    brandon,2,geographical name
    single,2,adjective
    player,2,noun
    while,1,noun
    in,1,preposition
    top,1,noun
    team,1,noun
    latest,1,adjective
    gem,1,noun
    catch,1,verb
    rob,1,verb
    base,1,noun
    hit,1,verb
    home,1,noun
    run,1,verb
    it,1,pronoun
    your,1,adjective
    call,1,verb
    vote,1,noun
    now,1,adverb
    total,2,adjective
    blue,1,adjective
    follow,2,verb
    baseball,2,noun
    tonight,2,adverb
    twitter,2,verb
    http,1,abbreviation
    www,1,abbreviation
    com,1,abbreviation
    mobile,2,adjective
    option,2,noun
    task,1,noun
    code,1,noun
    air,1,noun
    thursday,2,noun
    december,3,noun
    john,1,noun
    aaron,2,noun
    cruz,1,biographical name
    jr,1,abbreviation
    mark,1,noun
    singleton,3,noun
    join,1,verb
    us,1,pronoun
    nfl,1,abbreviation
    nba,1,abbreviation
    nhl,1,abbreviation
    nascar,1,abbreviation
    soccer,2,noun
    tennis,2,noun
    boxing,2,noun
    mma,1,abbreviation
    insider,3,noun
    sn,1,Symbol
    radio,2,adjective
    playbook,2,noun
    fantasy,3,noun
    watch,1,verb
    schedule,2,noun
    east,1,adverb
    baltimore,3,biographical name
    red,1,adjective
    tampa,2,geographical name
    bay,1,adjective
    central,2,adjective
    white,1,adjective
    cleveland,2,biographical name
    detroit,2,geographical name
    kansas,2,geographical name
    city,1,noun
    minnesota,4,geographical name
    west,1,adverb
    houston,2,biographical name
    oakland,2,geographical name
    seattle,3,geographical name
    texas,2,noun
    atlanta,3,geographical name
    miami,2,noun
    philadelphia,4,geographical name
    washington,3,biographical name
    cincinnati,3,geographical name
    pittsburgh,2,geographical name
    st,1,abbreviation
    louis,2,biographical name
    arizona,4,geographical name
    san,1,noun
    hall,1,noun
    of,1,preposition
    help,1,verb
    press,1,noun
    advertise,3,verb
    sales,1,adjective
    media,2,noun
    kit,1,noun
    interest,3,noun
    contact,2,noun
    site,1,noun
    map,1,noun
    jobs,1,biographical name
    at,1,preposition
    information,4,noun
    internet,3,noun
    use,1,noun
    privacy,3,noun
    policy,3,noun
    safety,2,noun
    california,4,geographical name
    applicable,4,adjective
    you,1,pronoun
    all,1,adjective
    reserved,2,adjective

`get-perl.csv`:

    perl,1,biographical name
    download,2,noun
    www,1,abbreviation
    org,1,abbreviation
    home,1,noun
    documentation,5,noun
    community,4,noun
    get,1,verb
    about,1,adverb
    may,1,verbal auxiliary
    not,1,adverb
    be,1,verb
    on,1,preposition
    over,1,adverb
    we,1,pronoun, plural in construction
    that,1,pronoun
    you,1,pronoun
    always,2,adverb
    run,1,verb
    the,1,definite article
    version,2,noun
    if,1,conjunction
    re,1,noun
    a,1,noun
    than,1,conjunction
    find,1,verb
    of,1,preposition
    will,1,verb
    work,1,noun
    information,4,noun
    running,1,noun
    or,1,conjunction
    any,1,adjective
    other,2,adjective
    like,1,verb
    system,2,noun
    already,2,adverb
    have,1,verb
    line,1,noun
    to,1,preposition
    out,1,adverb
    which,1,adjective
    binary,3,noun
    for,1,preposition
    many,1,adjective
    this,1,pronoun
    install,2,transitive verb
    source,1,noun
    latest,1,adjective
    stable,2,noun
    consider,3,verb
    at,1,preposition
    help,1,verb
    and,1,conjunction
    manage,2,verb
    from,1,preposition
    more,1,adjective
    code,1,noun
    development,4,noun
    as,1,adverb
    well,1,noun
    current,2,adjective
    under,2,adverb
    open,1,adjective
    in,1,preposition
    your,1,adjective
    by,1,preposition
    same,1,adjective
    need,1,noun
    available,2,adjective
    win,1,verb
    see,1,verb
    through,1,preposition
    it,1,pronoun
    include,2,transitive verb
    useful,2,adjective
    possible,3,adjective
    even,1,noun
    with,1,preposition
    learn,1,verb
    also,2,adverb
    support,2,transitive verb
    commercial,3,adjective
    when,1,adverb
    best,1,adjective
    enough,1,adjective
    sponsor,2,noun
    search,1,verb
    site,1,noun
    info,2,noun

`nfl-blog.csv`:

    more,1,adjective
    asia,1,geographical name
    united,1,adjective
    new,1,adjective
    blog,1,noun
    home,1,noun
    north,1,adverb
    west,1,adverb
    the,1,definite article
    january,3,noun
    by,1,preposition
    com,1,abbreviation
    recommend,3,transitive verb
    print,1,noun
    on,1,preposition
    i,1,noun
    a,1,noun
    post,1,noun
    in,1,preposition
    which,1,adjective
    it,1,pronoun
    time,1,noun
    for,1,preposition
    dallas,2,biographical name
    to,1,preposition
    start,1,verb
    some,1,adjective
    level,2,noun
    and,1,conjunction
    his,1,adjective
    way,1,noun
    that,1,pronoun
    team,1,noun
    its,1,adjective
    future,2,adjective
    of,1,preposition
    with,1,preposition
    from,1,preposition
    possible,3,adjective
    as,1,adverb
    we,1,pronoun, plural in construction
    have,1,verb
    this,1,pronoun
    change,1,verb
    since,1,adverb
    may,1,verbal auxiliary
    need,1,noun
    cut,1,verb
    like,1,verb
    or,1,conjunction
    would,1,verb
    be,1,verb
    though,1,conjunction
    not,1,adverb
    one,1,adjective
    will,1,verb
    made,1,adjective
    an,1,indefinite article
    all,1,adjective
    can,1,verb
    hope,1,verb
    make,1,verb
    because,2,conjunction
    they,1,pronoun, plural in construction
    get,1,verb
    only,2,adjective
    who,1,pronoun
    first,1,adjective
    your,1,adjective
    pray,1,verb
    what,1,pronoun
    you,1,pronoun
    next,1,adjective
    at,1,preposition
    switch,1,noun
    even,1,noun
    good,1,adjective
    never,2,adverb
    how,1,adverb
    many,1,adjective
    times,1,preposition
    if,1,conjunction
    look,1,verb
    kind,1,noun
    people,2,noun
    run,1,verb
    there,1,adverb
    however,3,conjunction
    come,1,verb
    when,1,adverb
    explain,2,verb
    their,1,adjective
    why,1,adverb
    think,1,verb
    significant,4,adjective
    enough,1,adjective
    should,1,verbal auxiliary
    now,1,adverb
    my,1,adjective
    end,1,noun
    up,1,adverb
    replace,2,transitive verb
    such,1,adjective
    second,2,adjective
    year,1,noun
    pass,1,verb
    but,1,conjunction
    also,2,adverb
    bruce,1,biographical name
    carter,2,biographical name
    want,1,verb
    them,1,pronoun
    don,1,transitive verb
    re,1,noun
    fan,1,noun
    part,1,noun
    whole,1,adjective
    best,1,adjective
    do,1,verb
    paul,1,noun
    help,1,verb
    place,1,noun
    report,2,noun
    today,2,adverb
    transition,3,noun
    current,2,adjective
    titled,1,adjective
    could,1,verbal auxiliary
    he,1,pronoun
    direct,2,verb
    offer,2,verb
    jump,1,verb
    work,1,noun
    matter,2,noun
    much,1,adjective
    life,1,noun
    else,1,adverb
    about,1,adverb
    probably,3,adverb
    seven,2,noun
    experience,4,noun
    most,1,adjective
    chicago,3,geographical name
    york,1,biographical name
    here,1,adverb
    period,3,noun
    idea,1,noun
    land,1,noun
    well,1,noun
    general,3,adjective
    great,1,adjective
    season,2,noun
    combined,2,noun
    long,1,adjective
    very,1,adjective
    position,3,noun
    last,1,verb
    without,2,preposition
    spirit,2,noun
    miss,1,verb
    journey,2,noun
    head,1,noun
    coach,1,noun
    better,2,adjective
    always,2,adverb
    word,1,noun
    five,1,noun
    game,1,noun
    between,2,preposition
    see,1,verb
    line,1,noun
    slot,1,noun
    title,2,noun
    quarterback,3,noun
    defense,2,noun
    against,1,preposition
    read,1,verb
    information,4,noun
    running,1,noun
    than,1,conjunction
    before,2,adverb or adjective
    after,2,adverb
    history,3,noun
    then,1,adverb
    four,1,noun
    win,1,verb
    third,1,adjective
    final,2,adjective
    other,2,adjective
    both,1,pronoun, plural in construction
    two,1,adjective
    tight,1,adjective
    three,1,noun
    wide,1,adjective
    michael,2,noun
    become,2,verb
    over,1,adverb
    touchdown,2,noun
    road,1,noun
    championship,4,noun
    out,1,adverb
    era,1,noun
    lewis,2,noun
    linebacker,3,noun
    point,1,noun
    player,2,noun
    single,2,adjective
    effort,2,noun
    arm,1,noun
    high,1,adjective
    raven,2,noun
    th,1,abbreviation
    deep,1,adjective
    week,1,noun
    contain,2,verb
    where,1,adverb
    outside,2,noun
    setting,1,noun
    consistent,3,adjective
    pressure,2,noun
    fewer,2,pronoun, plural in construction
    among,1,preposition
    john,1,noun
    tie,1,noun
    each,1,adjective
    conference,3,noun
    same,1,adjective
    join,1,verb
    total,2,adjective
    into,2,preposition
    during,2,preposition
    rank,1,adjective
    list,1,verb
    represent,3,verb
    play,1,noun
    twice,1,adverb
    note,1,transitive verb
    hall,1,noun
    suggest,2,transitive verb
    st,1,abbreviation
    target,2,noun
    count,1,verb
    while,1,noun
    through,1,preposition
    set,1,verb
    offense,2,noun
    rush,1,noun
    source,1,noun
    thanks,1,noun plural
    often,2,adverb
    formation,3,noun
    beginning,3,noun
    leading,2,adjective
    yard,1,noun
    average,3,noun
    mark,1,noun
    close,1,verb
    several,3,adjective
    respectively,4,adverb
    rarely,2,adverb
    lose,1,verb
    previous,3,adjective
    special,2,adjective
    return,2,verb
    aim,1,verb
    must,1,verb
    known,1,adjective
    expect,2,verb
    use,1,noun
    key,1,noun
    tempo,2,noun
    early,2,adverb
    particularly,5,adverb
    keep,1,verb
    tough,1,adjective
    double,2,adjective
    classic,2,adjective
    try,1,verb
    take,1,verb
    enter,2,verb
    injury,3,noun
    improvement,3,noun
    potential,3,adjective
    ken,1,verb
    rule,1,noun
    tried,1,adjective
    interview,3,noun
    another,3,adjective
    any,1,adjective
    move,1,verb
    given,2,adjective
    focus,2,noun
    area,2,noun
    style,1,noun
    sometimes,2,adverb
    september,3,noun
    approach,2,verb
    neither,2,conjunction
    mean,1,verb
    consider,3,verb
    project,2,noun
    add,1,verb
    quality,3,noun
    under,2,adverb
    worked,1,adjective
    theme,1,noun
    strong,1,adjective
    short,1,adjective
    say,1,verb
    president,3,noun
    group,1,noun
    collaborate,4,intransitive verb
    news,1,noun plural but singular in construction
    toward,2,adjective
    relationship,4,noun
    exist,2,intransitive verb
    critical,3,adjective
    draft,1,noun
    tweet,1,noun
    involved,1,adjective
    pro,1,noun
    george,1,noun
    mix,1,verb
    understanding,4,noun
    create,2,verb
    culture,2,noun
    making,1,noun
    manager,3,noun
    david,2,noun
    talk,1,verb
    me,1,pronoun
    little,2,adjective
    together,3,adverb
    fast,1,adjective
    city,1,noun
    name,1,noun
    birth,1,noun
    saint,1,noun
    intended,1,adjective
    call,1,verb
    actually,4,adverb
    briefly,2,adverb
    trip,1,verb
    young,1,adjective
    important,3,adjective
    find,1,verb
    scheme,1,noun
    understand,3,verb
    us,1,pronoun
    multiple,3,adjective
    cause,1,noun
    produce,2,verb
    being,2,noun
    simply,2,adverb
    every,2,adjective
    day,1,noun
    fall,1,verb
    maurice,2,biographical name
    te,1,symbol
    exactly,3,adverb
    april,1,noun
    sport,1,verb
    media,2,noun
    fill,1,verb
    go,1,verb
    overall,2,adverb
    obviously,4,adverb
    available,2,adjective
    chase,1,noun
    missing,2,adjective
    problem,2,noun
    already,2,adverb
    seem,1,intransitive verb
    quick,1,adjective
    nice,1,adjective
    stay,1,noun
    spot,1,noun
    pick,1,verb
    sense,1,noun
    london,2,biographical name
    either,2,adjective
    late,1,adjective
    de,1,abbreviation
    again,1,adverb
    old,1,adjective
    selected,3,adjective
    chief,1,adjective
    still,1,adjective
    value,2,noun
    august,2,adjective
    comment,2,noun
    lazy,2,adjective
    moment,2,noun
    notable,3,adjective
    latter,2,adjective
    category,4,noun
    series,2,noun
    member,2,noun
    lead,1,verb
    give,1,verb
    later,1,adverb
    deal,1,noun
    whose,1,adjective
    base,1,noun
    power,2,noun
    perception,3,noun
    subject,2,noun
    raised,1,adjective
    used,1,adjective
    depend,2,intransitive verb
    own,1,adjective
    back,1,noun
    top,1,noun
    fox,1,noun
    subscribe,2,verb
    rss,1,noun
    follow,2,verb
    archive,2,noun
    select,2,adjective
    december,3,noun
    october,3,noun
    july,2,noun
    june,1,noun
    march,1,noun
    february,3,noun
    afc,1,abbreviation
    nfc,1,abbreviation
    regular,3,adjective
    free,1,adjective
    around,1,adverb
    big,1,adjective
    boston,2,noun
    nick,1,noun
    reaction,3,noun
    louis,2,biographical name
    cardinal,3,noun
    texas,2,noun
    network,2,noun
    links,1,noun plural
    recent,2,adjective
    search,1,verb
    watch,1,verb
    center,2,noun
    weekly,2,adverb
    scouting,2,noun
    local,2,adjective
    press,1,noun
    kit,1,noun
    interest,3,noun
    contact,2,noun
    site,1,noun
    map,1,noun
    jobs,1,biographical name

`nhl-blog.csv`:

    usa,1,abbreviation
    more,1,adjective
    asia,1,geographical name
    australia,3,geographical name
    brazil,2,geographical name
    united,1,adjective
    kingdom,2,noun
    boston,2,noun
    chicago,3,geographical name
    dallas,2,biographical name
    los,1,abbreviation
    new,1,adjective
    york,1,biographical name
    espn,1,abbreviation
    nhl,1,abbreviation
    shop,1,noun
    cross,1,noun
    blog,1,noun
    home,1,noun
    draft,1,noun
    january,3,noun
    jan,1,abbreviation
    pm,1,abbreviation
    recommend,3,transitive verb
    tweet,1,noun
    print,1,noun
    from,1,preposition
    the,1,definite article
    release,2,verb
    agree,1,verb
    to,1,preposition
    a,1,noun
    deal,1,noun
    in,1,preposition
    with,1,preposition
    st,1,abbreviation
    louis,2,biographical name
    executive,4,adjective
    president,3,noun
    and,1,conjunction
    general,3,adjective
    manager,3,noun
    club,1,noun
    principal,3,adjective
    defenseman,3,noun
    today,2,adverb
    of,1,preposition
    contract,2,noun
    for,1,preposition
    one,1,adjective
    year,1,noun
    solid,2,adjective
    two,1,adjective
    way,1,noun
    we,1,pronoun, plural in construction
    believe,2,verb
    his,1,adjective
    experience,4,noun
    will,1,verb
    add,1,verb
    our,1,adjective
    defensive,3,adjective
    spent,1,adjective
    past,1,adjective
    american,3,noun
    league,1,noun
    connecticut,4,geographical name
    last,1,verb
    he,1,pronoun
    four,1,noun
    pound,1,noun
    most,1,adjective
    recently,3,adverb
    overall,2,adverb
    national,3,adjective
    penalty,3,noun
    by,1,preposition
    first,1,adjective
    round,1,adjective
    nd,1,abbreviation
    entry,2,noun
    this,1,pronoun
    on,1,preposition
    sunday,2,noun
    season,2,noun
    forecast,2,verb
    competitive,4,adjective
    balance,2,noun
    recent,2,adjective
    building,2,noun
    upon,2,preposition
    it,1,pronoun
    that,1,pronoun
    go,1,verb
    down,1,adverb
    regular,3,adjective
    final,2,adjective
    day,1,noun
    post,1,noun
    match,1,noun
    each,1,adjective
    into,2,preposition
    game,1,noun
    over,1,adverb
    days,1,adverb
    be,1,verb
    lot,1,noun
    eight,1,noun
    different,3,adjective
    have,1,verb
    cup,1,noun
    team,1,noun
    as,1,adverb
    since,1,adverb
    half,1,noun
    advanced,2,adjective
    conference,3,noun
    five,1,noun
    six,1,noun
    come,1,verb
    west,1,adverb
    division,3,noun
    title,2,noun
    an,1,indefinite article
    average,3,noun
    turnover,2,noun
    ross,1,biographical name
    every,2,adjective
    third,1,adjective
    when,1,adverb
    lost,1,adjective
    schedule,2,noun
    still,1,adjective
    up,1,adverb
    seven,2,noun
    yet,1,adverb
    potential,3,adjective
    series,2,noun
    why,1,adverb
    not,1,adverb
    us,1,pronoun
    seed,1,noun
    second,2,adjective
    best,1,adjective
    championship,4,noun
    run,1,verb
    all,1,adjective
    ever,2,adverb
    reach,1,verb
    win,1,verb
    sixth,1,noun
    advance,2,verb
    finished,2,adjective
    th,1,abbreviation
    during,2,preposition
    outside,2,noun
    top,1,noun
    interest,3,noun
    times,1,preposition
    launch,1,verb
    banner,2,noun
    regional,3,adjective
    coverage,3,noun
    at,1,preposition
    or,1,conjunction
    pittsburgh,2,geographical name
    philadelphia,4,geographical name
    night,1,noun
    select,2,adjective
    tonight,2,adverb
    toronto,3,geographical name
    network,2,noun
    washington,3,biographical name
    san,1,noun
    begin,2,verb
    friday,2,noun
    everyone,3,pronoun
    month,1,noun
    ray,1,noun
    detroit,2,geographical name
    weekend,2,noun
    america,3,geographical name
    your,1,adjective
    try,1,verb
    free,1,adjective
    local,2,adjective
    live,1,verb
    buffalo,3,noun
    city,1,noun
    classic,2,adjective
    dame,1,noun
    miami,2,noun
    ohio,1,geographical name
    minnesota,4,geographical name
    wisconsin,3,geographical name
    field,1,noun
    march,1,noun
    ryan,2,biographical name
    wild,1,adjective
    april,1,noun
    trade,1,noun
    thursday,2,noun
    energy,3,noun
    center,2,noun
    carolina,4,geographical name
    jordan,2,biographical name
    may,1,verbal auxiliary
    through,1,preposition
    june,1,noun
    combine,2,verb
    deadline,2,noun
    possible,3,adjective
    date,1,noun
    kickoff,2,noun
    com,1,abbreviation
    enough,1,adjective
    must,1,verb
    if,1,conjunction
    really,3,adverb
    decide,2,verb
    who,1,pronoun
    should,1,verbal auxiliary
    next,1,adjective
    their,1,adjective
    name,1,noun
    well,1,noun
    you,1,pronoun
    look,1,verb
    find,1,verb
    made,1,adjective
    pretty,2,adjective
    much,1,adjective
    already,2,adverb
    set,1,verb
    only,2,adjective
    after,2,adverb
    spot,1,noun
    i,1,noun
    bet,1,noun
    minded,2,adjective
    fall,1,verb
    bottom,2,noun
    western,2,adjective
    now,1,adverb
    there,1,adverb
    many,1,adjective
    which,1,adjective
    big,1,adjective
    some,1,adjective
    but,1,conjunction
    check,1,noun
    out,1,adverb
    long,1,adjective
    florida,3,geographical name
    tampa,2,geographical name
    bay,1,adjective
    red,1,adjective
    colorado,4,geographical name
    blue,1,adjective
    notable,3,adjective
    information,4,noun
    george,1,noun
    jason,2,noun
    aaron,2,noun
    tim,1,abbreviation
    thomas,2,noun
    joe,1,noun
    scott,1,biographical name
    brandon,2,geographical name
    jay,1,noun
    nick,1,noun
    rick,1,noun
    marc,1,noun
    taylor,2,biographical name
    cam,1,noun
    mason,2,noun
    matt,1,abbreviation
    stay,1,noun
    james,1,noun
    van,1,noun
    goal,1,noun
    needs,1,adverb
    career,2,noun
    assist,2,verb
    point,1,noun
    make,1,verb
    room,1,noun
    clear,1,adjective
    signal,2,noun
    decision,3,noun
    put,1,verb
    time,1,noun
    young,1,adjective
    like,1,verb
    get,1,verb
    old,1,adjective
    once,1,adverb
    exciting,3,adjective
    player,2,noun
    impact,2,verb
    they,1,pronoun, plural in construction
    him,1,pronoun
    summer,2,noun
    tough,1,adjective
    give,1,verb
    coach,1,noun
    start,1,verb
    nothing,2,pronoun
    gm,1,abbreviation
    good,1,adjective
    forward,2,adjective
    change,1,verb
    part,1,noun
    organization,5,noun
    could,1,verbal auxiliary
    here,1,adverb
    even,1,noun
    though,1,conjunction
    short,1,adjective
    certainly,3,adverb
    better,2,adjective
    system,2,noun
    about,1,adverb
    group,1,noun
    would,1,verb
    chance,1,noun
    winning,1,noun
    don,1,transitive verb
    doing,2,noun
    any,1,adjective
    high,1,adjective
    value,2,noun
    sound,1,adjective
    given,2,adjective
    both,1,pronoun, plural in construction
    around,1,adverb
    reason,2,noun
    can,1,verb
    dan,1,noun
    former,2,adjective
    pick,1,verb
    key,1,noun
    element,3,noun
    before,2,adverb or adjective
    off,1,adverb
    going,2,noun
    fill,1,verb
    opportunity,5,noun
    place,1,noun
    line,1,noun
    able,1,adjective
    various,3,adjective
    hockey,2,noun
    seventh,2,noun
    than,1,conjunction
    prior,2,noun
    important,3,adjective
    filling,2,noun
    play,1,noun
    together,3,adverb
    power,2,noun
    how,1,adverb
    used,1,adjective
    whether,2,pronoun
    being,2,noun
    publicly,3,adverb
    guy,1,noun
    coming,2,noun
    replace,2,transitive verb
    three,1,noun
    generally,4,adverb
    something,2,pronoun
    afraid,1,adjective
    obviously,4,adverb
    kind,1,noun
    situation,4,noun
    people,2,noun
    calling,2,noun
    performance,3,noun
    where,1,adverb
    great,1,adjective
    job,1,noun
    very,1,adjective
    successful,3,adjective
    far,1,adverb
    think,1,verb
    success,2,noun
    do,1,verb
    ll,1,abbreviation
    re,1,noun
    whatever,3,pronoun
    need,1,noun
    me,1,pronoun
    trend,1,intransitive verb
    let,1,transitive verb
    break,1,verb
    also,2,adverb
    full,1,adjective
    throughout,2,adverb
    front,1,noun
    back,1,noun
    rule,1,noun
    its,1,adjective
    right,1,adjective
    my,1,adjective
    agent,1,noun
    against,1,preposition
    cap,1,noun
    possibility,5,noun
    related,1,adjective
    smart,1,adjective
    move,1,verb
    heck,1,noun
    sure,1,adjective
    respect,2,noun
    strong,1,adjective
    franchise,2,noun
    string,1,noun
    until,2,preposition
    hour,1,noun
    agency,2,noun
    memory,3,noun
    fresh,1,adjective
    mind,1,noun
    perspective,3,noun
    see,1,verb
    mean,1,verb
    term,1,noun
    say,1,verb
    depend,2,intransitive verb
    position,3,noun
    fact,1,noun
    understanding,4,noun
    actually,4,adverb
    truly,2,adverb
    max,1,noun
    speaking,1,adjective
    too,1,adverb
    away,1,adverb
    affect,2,noun
    ability,3,noun
    star,1,noun
    july,2,noun
    while,1,noun
    younger,2,noun
    comparison,4,noun
    trying,1,adjective
    matter,2,noun
    early,2,adverb
    always,2,adverb
    father,2,noun
    deciding,3,adjective
    retire,2,verb
    take,1,verb
    popular,3,adjective
    longtime,2,adjective
    draw,1,verb
    week,1,noun
    morning,2,noun
    thinking,1,noun
    south,1,adverb
    heading,2,noun
    open,1,adjective
    anything,2,pronoun
    decided,3,adjective
    likely,2,adjective
    few,1,pronoun, plural in construction
    ago,1,adjective or adverb
    couple,2,noun
    easy,1,adjective
    thing,1,noun
    want,1,verb
    fine,1,noun
    anymore,2,adverb
    case,1,noun
    jobs,1,biographical name
    tight,1,adjective
    roster,2,noun
    evaluate,3,transitive verb
    help,1,verb
    either,2,adjective
    know,1,verb
    then,1,adverb
    plus,1,adjective
    under,2,adverb
    among,1,preposition
    house,1,noun
    resonate,3,verb
    them,1,pronoun
    emotional,3,adjective
    california,4,geographical name
    total,2,adjective
    between,2,preposition
    met,1,noun,
    special,2,adjective
    what,1,pronoun
    road,1,noun
    work,1,noun
    spring,1,noun
    perhaps,2,adverb
    scouting,2,noun
    rush,1,noun
    figure,2,noun
    involved,1,adjective
    form,1,noun
    another,3,adjective
    ride,1,verb
    reporter,3,noun
    opening,2,noun
    question,2,noun
    talk,1,verb
    feel,1,verb
    end,1,noun
    entire,2,adjective
    although,2,conjunction
    across,1,adverb
    phone,1,noun
    watch,1,verb
    expectation,4,noun
    other,2,adjective
    kid,1,noun
    pressure,2,noun
    fair,1,adjective
    answer,2,noun
    foot,1,noun
    behind,2,adverb or adjective
    nice,1,adjective
    bit,1,noun
    business,2,noun
    especially,4,adverb
    world,1,noun
    goaltending,3,noun
    knowing,1,noun
    least,1,adjective
    emerge,1,intransitive verb
    crack,1,verb
    search,1,verb
    jonathan,3,noun
    option,2,noun
    pair,1,noun
    account,2,noun
    per,1,preposition
    structure,2,noun
    carry,2,verb
    never,2,adverb
    step,1,noun
    prove,1,verb
    comfortable,3,adjective
    backup,2,noun
    role,1,noun
    personality,5,noun
    life,1,noun
    without,2,preposition
    else,1,adverb
    era,1,noun
    ken,1,verb
    european,4,adjective
    guide,1,noun
    record,2,verb
    book,1,noun
    defense,2,noun
    keeping,1,noun
    elaborate,3,adjective
    matchup,2,noun
    white,1,adjective
    smith,1,noun
    bad,1,adjective
    lose,1,verb
    quality,3,noun
    mark,1,noun
    jimmy,2,noun
    howard,2,biographical name
    source,1,noun
    sometimes,2,adverb
    story,2,noun
    obvious,3,adjective
    absolutely,4,adverb
    lineup,2,noun
    ethic,2,noun
    hole,1,noun
    person,2,noun
    again,1,adverb
    reality,4,noun
    simply,2,adverb
    level,2,noun
    seem,1,intransitive verb
    definition,4,noun
    concept,2,noun
    david,2,noun
    addition,3,noun
    invite,2,transitive verb
    missing,2,adjective
    probably,3,adverb
    en,1,noun
    style,1,noun
    interview,3,noun
    healthy,1,adjective
    main,1,noun
    offensive,3,adjective
    keep,1,verb
    tom,1,noun
    noted,1,adjective
    offense,2,noun
    notion,2,noun
    score,1,noun
    appreciation,5,noun
    improve,2,verb
    hot,1,adjective
    fantasy,3,noun
    news,1,noun plural but singular in construction
    side,1,noun
    talent,2,noun
    translate,2,verb
    flourish,2,verb
    pro,1,noun
    rival,2,noun
    attention,3,noun
    wing,1,noun
    same,1,adjective
    vernon,2,biographical name
    columbia,3,noun
    jump,1,verb
    headed,2,adjective
    offer,2,verb
    ultimately,4,adverb
    seriously,4,adverb
    become,2,verb
    worked,1,adjective
    quick,1,adjective
    less,1,adjective
    create,2,verb
    largely,2,adverb
    because,2,conjunction
    listen,2,verb
    stuff,1,noun
    below,2,adverb
    learn,1,verb
    adjustment,3,noun
    period,3,noun
    flying,1,adjective
    quite,1,adverb
    hit,1,verb
    rest,1,noun
    late,1,adjective
    nearly,2,adverb
    show,1,verb
    accurate,3,adjective
    acquire,2,transitive verb
    whole,1,adjective
    simple,2,adjective
    rebound,2,verb
    candidate,3,noun
    peg,1,noun
    following,3,adjective
    process,2,noun
    learning,1,noun
    consistent,3,adjective
    love,1,noun
    expect,2,verb
    track,1,noun
    little,2,adjective
    sweden,2,geographical name
    beginning,3,noun
    paul,1,noun
    jones,1,noun
    john,1,noun
    unit,1,noun
    face,1,noun
    type,1,noun
    bring,1,verb
    leading,2,adjective
    return,2,verb
    continue,3,verb
    whenever,3,conjunction
    somewhat,2,pronoun
    ending,2,noun
    injury,3,noun
    broken,2,adjective
    call,1,verb
    panic,2,adjective
    sake,1,noun
    moving,1,adjective
    media,2,noun
    driven,1,adjective
    relationship,4,noun
    hope,1,verb
    soon,1,adverb
    touched,1,adjective
    willing,2,adjective
    view,1,noun
    wake,1,verb
    announcement,3,noun
    association,5,noun
    discuss,2,transitive verb
    unable,2,adjective
    plan,1,noun
    use,1,noun
    count,1,verb
    neither,2,conjunction
    nor,1,conjunction
    thus,1,adverb
    qualify,3,verb
    hurt,1,verb
    future,2,adjective
    hard,1,adjective
    happen,2,intransitive verb
    common,2,adjective
    sense,1,noun
    allow,2,verb
    issue,2,noun
    solution,3,noun
    page,1,noun
    subscribe,2,verb
    rss,1,noun
    archive,2,noun
    december,3,noun
    october,3,noun
    september,3,noun
    august,2,adjective
    february,3,noun
    atlanta,3,geographical name
    joining,2,noun
    co,1,abbreviation
    follow,2,verb
    twitter,2,verb
    press,1,noun
    real,2,adjective
    send,1,verb
    cover,2,verb
    magazine,3,noun
    insider,3,noun
    journal,2,noun
    net,1,noun
    commercial,3,adjective
    famous,2,adjective
    http,1,abbreviation
    surgery,2,noun
    knee,1,noun
    baby,2,noun
    practice,2,verb
    join,1,verb
    preview,2,transitive verb
    chat,1,verb
    interesting,4,adjective
    hear,1,verb
    rookie,2,noun
    happy,2,adjective
    price,1,noun
    hence,1,adverb
    nfl,1,abbreviation
    mlb,1,abbreviation
    nba,1,abbreviation
    nascar,1,abbreviation
    soccer,2,noun
    tennis,2,noun
    boxing,2,noun
    mma,1,abbreviation
    sn,1,Symbol
    radio,2,adjective
    playbook,2,noun
    southeast,2,adverb
    central,2,adjective
    attendance,3,noun
    daily,2,adjective
    history,3,noun
    index,2,noun
    winter,2,noun
    hall,1,noun
    fame,1,noun
    advertise,3,verb
    sales,1,adjective
    kit,1,noun
    contact,2,noun
    site,1,noun
    map,1,noun
    internet,3,noun
    privacy,3,noun
    policy,3,noun
    safety,2,noun
    applicable,4,adjective
    reserved,2,adjective

`rakudo-star-release.csv`:

    org,1,abbreviation
    to,1,preposition
    content,2,adjective
    about,1,adverb
    community,4,noun
    how,1,adverb
    help,1,verb
    on,1,preposition
    by,1,preposition
    of,1,preposition
    the,1,definite article
    and,1,conjunction
    development,4,noun
    i,1,noun
    december,3,noun
    release,2,verb
    a,1,noun
    for,1,preposition
    from,1,preposition
    star,1,noun
    will,1,verb
    appear,2,intransitive verb
    in,1,preposition
    after,2,adverb
    world,1,noun
    we,1,pronoun, plural in construction
    make,1,verb
    between,2,preposition
    language,2,noun
    such,1,adjective
    as,1,adverb
    this,1,pronoun
    various,3,adjective
    documentation,5,noun
    other,2,adjective
    some,1,adjective
    new,1,adjective
    include,2,transitive verb
    follow,2,verb
    standard,2,noun
    more,1,adjective
    they,1,pronoun, plural in construction
    given,2,adjective
    now,1,adverb
    parse,1,verb
    an,1,indefinite article
    order,2,verb
    give,1,verb
    perl,1,biographical name
    considered,3,adjective
    not,1,adverb
    before,2,adverb or adjective
    also,2,adverb
    range,1,noun
    bug,1,noun
    error,2,noun
    better,2,adjective
    failure,2,noun
    following,3,adjective
    have,1,verb
    or,1,conjunction
    previous,3,adjective
    being,2,noun
    only,2,adjective
    eager,2,adjective
    context,2,noun
    that,1,pronoun
    loop,1,noun
    statement,2,noun
    it,1,pronoun
    return,2,verb
    into,2,preposition
    change,1,verb
    them,1,pronoun
    equivalent,3,adjective
    one,1,adjective
    be,1,verb
    add,1,verb
    leading,2,adjective
    again,1,adverb
    capture,2,noun
    list,1,verb
    bind,1,verb
    directly,3,adverb
    can,1,verb
    use,1,noun
    which,1,adjective
    continue,3,verb
    there,1,adverb
    key,1,noun
    yet,1,adverb
    although,2,conjunction
    at,1,preposition
    than,1,conjunction
    online,2,adjective
    resource,2,noun
    known,1,adjective
    many,1,adjective
    feature,2,noun
    but,1,conjunction
    see,1,verb
    http,1,abbreviation
    links,1,noun plural
    example,3,noun
    reference,3,noun
    book,1,noun
    team,1,noun
    all,1,adjective
    if,1,conjunction
    you,1,pronoun
    would,1,verb
    like,1,verb
    contribute,3,verb
    us,1,pronoun
    search,1,verb
    recent,2,adjective
    simon,2,noun
    september,3,noun
    june,1,noun
    may,1,verbal auxiliary
    april,1,noun
    february,3,noun
    meta,2,geographical name
    rss,1,noun
    skip,1,verb
    home,1,noun
    get,1,verb
    behalf,2,noun
    happy,2,adjective
    announce,2,verb
    useful,2,adjective
    usable,2,adjective
    distribution,4,noun
    available,2,adjective
    download,2,noun
    section,2,noun
    version,2,noun
    area,2,noun
    shortly,2,adverb
    distinction,3,noun
    specific,3,adjective
    compiler,3,noun
    parrot,2,noun
    virtual,3,adjective
    machine,2,noun
    collected,3,adjective
    much,1,adjective
    std,1,abbreviation
    parser,2,noun
    accurate,3,adjective
    information,4,noun
    less,1,adjective
    serious,3,adjective
    junction,2,noun
    magnitude,3,noun
    texas,2,noun
    ascii,1,noun
    set,1,verb
    bag,1,noun
    correct,2,transitive verb
    output,2,noun
    block,1,noun
    hash,1,transitive verb
    performance,3,noun
    due,1,adjective
    specification,5,noun
    removed,1,adjective
    become,2,verb
    lazy,2,adjective
    sink,1,verb
    void,1,adjective
    last,1,verb
    routine,2,noun
    run,1,verb
    call,1,verb
    anymore,2,adverb
    unary,2,adjective
    hyper,2,adjective
    ops,1,noun
    descend,2,verb
    level,2,noun
    map,1,noun
    str,1,abbreviation
    tc,1,symbol
    under,2,adverb
    expect,2,verb
    conversion,3,noun
    front,1,noun
    quantifier,4,noun
    slot,1,noun
    either,2,adjective
    zero,2,noun
    match,1,noun
    future,2,adjective
    nil,1,noun
    code,1,noun
    manage,2,verb
    transition,3,noun
    handle,2,noun
    upcoming,3,adjective
    quite,1,adverb
    advanced,2,adjective
    concurrency,4,noun
    interactive,4,adjective
    synopsis,3,noun
    missing,2,adjective
    tried,1,adjective
    smart,1,adjective
    enough,1,adjective
    inform,2,verb
    programmer,3,noun
    broken,2,adjective
    draft,1,noun
    pdf,1,noun
    thanks,1,noun plural
    making,1,noun
    possible,3,adjective
    ask,1,verb
    mailing,2,noun
    join,1,verb
    entry,2,noun
    bookmark,2,noun
    closed,1,adjective
    weekly,2,adverb
    maven,2,noun
    consistent,3,adjective
    de,1,abbreviation
    michael,2,noun
    october,3,noun
    august,2,adjective
    july,2,noun
    january,3,noun
    log,1,noun
    rakudo,3,noun

`uefa.csv`:

    accessible,4,adjective
    version,2,noun
    live,1,verb
    log,1,noun
    in,1,preposition
    your,1,adjective
    or,1,conjunction
    account,2,noun
    with,1,preposition
    com,1,abbreviation
    there,1,adverb
    to,1,preposition
    be,1,verb
    a,1,noun
    problem,2,noun
    the,1,definite article
    you,1,pronoun
    have,1,verb
    check,1,noun
    address,2,verb
    and,1,conjunction
    re,1,noun
    enter,2,verb
    choose,1,verb
    password,2,noun
    keep,1,verb
    me,1,pronoun
    signed,1,adjective
    more,1,adjective
    search,1,verb
    news,1,noun plural but singular in construction
    video,2,noun
    community,4,noun
    mobile,2,adjective
    about,1,adverb
    member,2,noun
    football,2,noun
    development,4,noun
    league,1,noun
    under,2,adverb
    all,1,adjective
    club,1,noun
    super,2,adjective
    cup,1,noun
    national,3,adjective
    world,1,noun
    youth,1,noun
    official,3,noun
    for,1,preposition
    european,4,adjective
    coming,2,noun
    up,1,adverb
    around,1,adverb
    away,1,adverb
    day,1,noun
    three,1,noun
    dutch,1,adverb
    between,2,preposition
    afc,1,abbreviation
    of,1,preposition
    on,1,preposition
    weekend,2,noun
    when,1,adverb
    united,1,adjective
    saint,1,noun
    real,2,adjective
    face,1,noun
    germany,3,geographical name
    netherlands,3,geographical name
    england,2,geographical name
    france,1,biographical name
    bar,1,noun
    st,1,abbreviation
    win,1,verb
    top,1,noun
    pick,1,verb
    week,1,noun
    good,1,adjective
    get,1,verb
    their,1,adjective
    due,1,adjective
    as,1,adverb
    year,1,noun
    spot,1,noun
    yet,1,adverb
    again,1,adverb
    we,1,pronoun, plural in construction
    last,1,verb
    seven,2,noun
    days,1,adverb
    team,1,noun
    follow,2,verb
    friday,2,noun
    travel,2,verb
    before,2,adverb or adjective
    host,1,noun
    long,1,adjective
    game,1,noun
    pep,1,noun
    match,1,noun
    general,3,adjective
    fight,1,verb
    which,1,adjective
    sport,1,verb
    integrity,4,noun
    at,1,preposition
    forum,2,noun
    chief,1,adjective
    technical,3,adjective
    officer,3,noun
    conference,3,noun
    links,1,noun plural
    president,3,noun
    executive,4,adjective
    committee,3,noun
    media,2,noun
    meeting,1,noun
    off,1,adverb
    financial,3,adjective
    fair,1,adjective
    play,1,noun
    head,1,noun
    january,3,noun
    press,1,noun
    popular,3,adjective
    talent,2,noun
    sweden,2,geographical name
    making,1,noun
    young,1,adjective
    free,1,adjective
    kick,1,verb
    killer,2,noun
    pass,1,verb
    next,1,adjective
    level,2,noun
    de,1,abbreviation
    latest,1,adjective
    new,1,adjective
    deal,1,noun
    he,1,pronoun
    sign,1,noun
    term,1,noun
    arsenal,3,noun
    forward,2,adjective
    now,1,adverb
    until,2,preposition
    summer,2,noun
    related,1,adjective
    out,1,adverb
    five,1,noun
    german,2,adjective
    lose,1,verb
    already,2,adverb
    one,1,adjective
    future,2,adjective
    but,1,conjunction
    still,1,adjective
    work,1,noun
    this,1,pronoun
    season,2,noun
    will,1,verb
    without,2,preposition
    rest,1,noun
    after,2,adverb
    brazil,2,geographical name
    surgery,2,noun
    knee,1,noun
    injury,3,noun
    other,2,adjective
    challenge,2,verb
    reflect,2,verb
    it,1,pronoun
    an,1,indefinite article
    great,1,adjective
    chosen,2,noun
    defender,3,noun
    his,1,adjective
    milan,2,geographical name
    competition,4,noun
    magazine,3,noun
    experience,4,noun
    key,1,noun
    family,3,noun
    march,1,noun
    target,2,noun
    player,2,noun
    i,1,noun
    jobs,1,biographical name
    archive,2,noun
    content,2,adjective
    sales,1,adjective
    network,2,noun
    twitter,2,verb
    rss,1,noun
    my,1,adjective
    change,1,verb
    language,2,noun
    english,2,adjective
    send,1,verb
    contact,2,noun
    us,1,pronoun
    help,1,verb
    privacy,3,noun
    reserved,2,adjective
    word,1,noun
    by,1,preposition
    trade,1,noun
    use,1,noun
    commercial,3,adjective
    may,1,verbal auxiliary
    made,1,adjective
    such,1,adjective
    agreement,2,noun
    policy,3,noun

`wikipedia.haiku.csv`:

    haiku,2,noun
    the,1,definite article
    this,1,pronoun
    a,1,noun
    from,1,preposition
    tradition,3,noun
    modern,2,adjective
    although,2,conjunction
    in,1,preposition
    see,1,verb
    on,1,preposition
    contemporary,4,adjective
    continue,3,verb
    one,1,adjective
    syllable,3,noun
    for,1,preposition
    count,1,verb
    unit,1,noun
    some,1,adjective
    phrase,1,noun
    poem,2,noun
    quality,3,noun
    example,3,noun
    contain,2,verb
    it,1,pronoun
    fuji,2,noun
    edo,1,geographical name
    by,1,preposition
    they,1,pronoun, plural in construction
    hokku,2,noun
    even,1,noun
    thus,1,adverb
    his,1,adjective
    he,1,pronoun
    famous,2,adjective
    poet,2,noun
    part,1,noun
    western,2,adjective
    sketch,1,noun
    influence,3,noun
    any,1,adjective
    since,1,adverb
    distinguish,3,verb
    there,1,adverb
    further,2,adverb
    early,2,adverb
    value,2,noun
    reader,2,noun
    at,1,preposition
    she,1,pronoun
    history,3,noun
    scholar,2,noun
    its,1,adjective
    nature,2,noun
    classic,2,adjective
    original,3,noun
    an,1,indefinite article
    anthology,4,noun
    after,2,adverb
    world,1,noun
    war,1,noun
    two,1,adjective
    imperial,4,adjective
    rhyme,1,noun
    however,3,conjunction
    because,2,conjunction
    depend,2,intransitive verb
    use,1,noun
    while,1,noun
    being,2,noun
    worldwide,2,adjective
    gathering,3,noun
    write,1,verb
    promote,2,transitive verb
    classical,3,adjective
    list,1,verb
    national,3,adjective
    like,1,verb
    winter,2,noun
    juxtaposition,5,noun
    cut,1,verb
    moment,2,noun
    journal,2,noun
    definition,4,noun
    literature,4,noun
    brief,1,adjective
    three,1,noun
    autumn,2,noun
    frog,1,noun
    commentary,3,noun
    of,1,preposition
    mt,1,abbreviation
    personal,3,adjective
    year,1,noun
    van,1,noun
    hundred,2,noun
    road,1,noun
    what,1,pronoun
    mean,1,verb
    read,1,verb
    linked,1,adjective
    verse,1,noun
    study,1,noun
    publishing,3,noun
    poetry,3,noun
    then,1,adverb
    volume,2,noun
    book,1,noun
    review,2,noun
    article,3,noun
    uk,1,abbreviation
    telegraph,2,noun
    europe,2,geographical name
    eu,1,symbol
    president,3,noun
    how,1,adverb
    share,1,noun
    landscape,2,noun
    cultural,3,adjective
    memory,3,noun
    ha,1,interjection
    poetic,3,adjective
    i,1,noun
    literary,3,adjective
