---
layout: post
title: Oh, hi there.
---
It's been just over a year since I updated this thing, so, figure it's about time to post something.

### Quick life update
The last year has been a whirlwind. Moved into my first big kid apartment in November, graduated with my BS in CS from SFSU in December, took my mom on my first trip overseas to Rome in February, adopted an adorable little rescue pibble named Isis in April, released records with both of my two bands (within a week of each other, actually) in September. The other day, one of my bands opened for one of my favorite bands at one of my favorite venues, so I'm going to be riding that high for a while. Life is fantastic.

### Quick work update
I'm still working at Grand Rounds on the Data Engineering team. We've been busy over the last quarter or so launching v2.0 of our insurance claims pipeline. The part of the project I'm responsible for is building a service that creates production-quality test data. I didn't intend to follow the fun "neighborhood" theme we've had going on with some of our recent repos, but my PM pointed out last week that calling my project the `laundromat` has done exactly that. (Whatever, it's cute.) I'm building it as a Ruby script (without Rails, which has been an interesting exercise) that reads from `STDIN`, scrubs each line of in file and replaces all the (PII)[https://en.wikipedia.org/wiki/Personally_identifiable_information] with fake names/addresses/etc., and then writes it back to `STDOUT` in the same format as the input. The purpose is to get us high quality data for testing our system without leaking any personal information from our patients. Huge shoutout to `ryanwood` for writing the (slither gem)[https://github.com/ryanwood/slither] because it's made my work about 300% easier; funny how a gem that hasn't been updated in 6+ years can still provide so much value.

I've also been practicing a lot of string/file/IO manipulation using the command line, so even though most of my work focus is Ruby-based, I figure it's probably a good idea to share some of the other fun things I learn here while I go.

---

## Using `diff` for sanity checks

For each insurance vendor's file format that we ingest, I need to build a JSON file that tells my `Laundry::Machine` (believe me, the analogy gets sillier as I go) what `Laundry::Detergents` are needed to `#scrub` each line of the input. My process for development is basically:

1. Create the JSON file and add whatever metadata I need to it (fixed width file vs. CSV, what the delimiters are for lines and columns, etc.).
2. List the column headers from the vendor's data dictionary.
3. Test to ensure I'm getting the correct in/output when not changing anything.
4. Indicate the transformations and test my in/output again to ensure the data still looks okay.
5. Generate a new test file and put it through our development pipeline to see if anything breaks.
6. Report any bugs to the rest of the team.
7. Rinse and repeat.

I mentioned earlier that the `laundromat` needs to write the same format as it reads in; this is important mostly because that's where Claims Pipeline v1.0 ran into many bugs as we built it. Line endings could be weird (thank you Windows for `\r\n`), whitespace could be important or unimportant, fixed width files are generally a pain in the rear, etc. The test files we built previously were hand-edited to remove PII, and text editors sometimes like to remove things that are actually really important to control for in our production system, so we ran into several occasions where a file would stop moving through the pipeline due to some special case we weren't expecting. Hence, when designing the `laundromat`, I wanted to ensure I'm getting the _exact_ same input and output when I'm not doing any transformations on the data. That's where `diff` comes in.

In step 3, I'll generally use one of our previously-created sample files, let's call it `$SAMPLE_FILE` for now. Now, I knew that `diff` could take two files as arguments, but what I didn't know is that it can also take the output of a command, like this:

```
$ diff <(command_one) <(command_two)
```

The `<()` is the bash-specific _substitution_ syntax, which passes the output of whatever is inside to the input of wherever you're directing it. It's handy when you can't use pipes, like with our friend `diff` up there.

Further, you can add all kinds of fun stuff inside the parentheses to dig deeper into whatever you're trying to do. Generally, though, I'll just start with a single line of my sample file:

```
$ diff <(head -1 $SAMPLE_FILE) <(head -1 $SAMPLE_FILE | ./laundromat.rb vendor_name)
```

If this produces no output, I can go on my merry way to step 4 of my dev cycle. If it does, though, it's time to start adding things into my diff command to figure out exactly what I missed. Usually the first thing from here is to pipe the `diff` output into `less` with the `-S` option, which will paginate the output horizontally, so I can (hopefully) visually see exactly where it's not lining up between the input and output.

```
$ diff <(head -1 $SAMPLE_FILE) <(head -1 $SAMPLE_FILE | ./laundromat.rb vendor_name) | less -S
```

If I don't see anything obvious right away, I'll add more stuff to figure out which field is giving me trouble. Since it's a CSV, I can replace the column delimiters with a newline and then run the diff on that.

```
$ diff <(head -1 $SAMPLE_FILE | tr , '\n') <(head -1 $SAMPLE_FILE | ./laundromat.rb vendor_name | tr , '\n')
```

If this **still** doesn't show me anything obvious, like in this example:

```
$ diff <(head -1 $SAMPLE_FILE | tr , '\n') <(head -1 $SAMPLE_FILE | ./laundromat.rb vendor_name | tr , '\n')
63c63
< "46912003601220"
---
> "46912003601220"
```

then I get to bust out another tool I love called `od`, which stands for "octal dump". Adding the `-c` option makes it show me the actual characters from the input, so it's more obvious what I'm looking at and I don't have to decipher which character `031466` stands for.

```
$ diff <(head -1 $SAMPLE_FILE | tr , '\n' | od -c) <(head -1 $SAMPLE_FILE | ./laundromat.rb vendor_name | tr , '\n' | od -c)
36,37c36,37
< 0001060    1   2   0   0   3   6   0   1   2   2   0   "  \r  \n        
< 0001076
---
> 0001060    1   2   0   0   3   6   0   1   2   2   0   "  \n            
> 0001075
```

And there we have our culprit--the freakin' Windows carriage return. (╯°□°）╯︵ ┻━┻ Now I can control for that in my script and head forward.

---

### Well, that was fun

I plan to publish more of these style of blog posts, because if nothing else they're pretty fun to write (and, I hope, more entertaining to read than StackOverflow). Stay tuned!
