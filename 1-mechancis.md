Ruby Command Line Fu - The Mechanics
====================================

One of Ruby's major influences was Perl, leaving well equipped to replace bash,
sed, and awk. Lets take a look at how to do that.


The `-e` flag
-------------

When you're composing a light-weight 1 off command line invocation, you don't want
to have to write your program in a file. The `-e` flag tells Ruby that the next
argument is your program.

```sh
$ ruby -e 'puts "hello world"'
hello world
```


The `-n` and `-p` flags, and `$_` variable
------------------------------------------

To replace sed and awk, we're going to need to iterate over every line of input.
The `-n` and `-p` flags do this. As they iterate, they place the current line in
the variable `$_`, then run your program. The difference is that `-p` will then
print `$_`.

```sh
$ echo -e "a\nb\nc" | ruby -n -e 'puts $_.upcase'
A
B
C

$ echo -e "a\nb\nc" | ruby -p -e '$_ = $_.upcase'
A
B
C
```

Exercise: Write `grep`
----------------------

Given a markdown file on stdin, print its headers. Assume they're using hash
style headers, not equal / dash style.

Here is an example input file:

```sh
$ curl -sL 'https://raw.githubusercontent.com/ruby/ruby/trunk/README.md'
```

Here is my solution, we'll soon learn ways to improve on it!

```sh
$ curl -sL 'https://raw.githubusercontent.com/ruby/ruby/trunk/README.md' | ruby -n -e 'puts $_ if $_[/^#+/]'
# What's Ruby
## Features of Ruby
## How to get Ruby
## Ruby home page
## Mailing list
## How to compile and install
## Copying
## Feedback
## Contributing
## The Author
```


Cache command line tool
-----------------------

So, the above might have been a bit painful since it can take over a second for
the curl to complete. I'm often in situations like that, where I need to
experiment with what to do next after running some expensive command.

That means I'll have to pay that expense a lot of times, which makes it cumbersome
to do the exploration. To avoid that expense, I wrote a tool called `cache`, which
runs a command line invocation, records its stdout/stderr/exit status, and then
replays them in future invocations. Once I've got the command-line invocation
figured out, I just remove the call to `cache`. You can get its source code [here](https://github.com/JoshCheek/dotfiles/blob/ee1607e3bf815a481c899627bcef04e5fedb926b/bin/cache).
Then put that somewhere in your path and `chmod +x` it. I usually put new / experimental command
line tools in `~/bin`, which I've added to my path in [bash](https://github.com/JoshCheek/dotfiles/blob/ee1607e3bf815a481c899627bcef04e5fedb926b/bash_profile#L3),
and [fish](https://github.com/JoshCheek/dotfiles/blob/ee1607e3bf815a481c899627bcef04e5fedb926b/fish/config.fish#L2).

You can use it like this:

```sh
$ time cache ruby -e 'sleep 1; puts "hello"'
hello
        1.31 real         0.16 user         0.05 sys

$ time cache ruby -e 'sleep 1; puts "hello"'
hello
        0.08 real         0.06 user         0.01 sys
```

So the way I dealt with the previous exercise was like this:

```sh
$ cache curl -sL 'https://raw.githubusercontent.com/ruby/ruby/trunk/README.md' \
>  | ruby -n -e 'puts $_ if $_[/^#+/]'
```

And then I removed the call to `cache` once I'd gotten it figured out.



Join flags to shorten the invocation
------------------------------------

When a short flag doesn't take args, you can join it with the next short flag,
here we do `-ne` instead of `-n -e`

```sh
$ echo -e "a\nb\nc" | ruby -ne 'puts $_.upcase'
A
B
C
```


`print` without args, prints `$_`
---------------------------------

```sh
echo -e "a\nb\nc" | ruby -ne 'print; print;'
a
a
b
b
c
c
```


Regex in a conditional matches against `$_`
-------------------------------------------

```sh
echo -e "a\nb\nc" | ruby -ne 'print if /[ab]/'
a
b
```


Improving our grep
------------------

Using these pieces, we can improve our grep!

```sh
# before
$ ruby -n -e 'puts $_ if $_[/^#+/]'

# after
$ ruby -ne 'print if /^#+/'
```


Shorthand interpolation
-----------------------

When a variable begins with a sigil, eg our cash symbol in `$_`, you can omit
the curly braces from the interpolation.

```ruby
$_ = "b"     # => "b"
"a #{$_} c"  # => "a b c"
"a #$_ c"    # => "a b c"
```

This does require that the next letter after its name not be a valid part of a
name. Eg `$_c` is a valid variable name, so we couldn't use the shorthand
interpolation for `"a#{$_}c"`.


The current line number: `$.`
-----------------------------

```sh
$ echo -e "a\nb\nc" | ruby -ne 'puts $., $_'
1
a
2
b
3
c
```


Exercise: Add line numbers to the input
---------------------------------------

Here is my solution:

```sh
$ echo -e "a\nb\nc" | ruby -ne 'puts "#$.\t#$_"'
1       a
2       b
3       c
```


`Regexp#~@` matches the regexp against stdin
--------------------------------------------

This is useful when Ruby isn't quite smart enough to figure out that the regex
is being used in a conditional.

```sh
$ ruby -ne 'print if /^#+/'
a
b
```


The `-l` flag performs newline processing
-----------------------------------------

It will chomp newlines from `$_`

```sh
$ echo -e "a\nb\nc" | ruby -ne 'p $_'
"a\n"
"b\n"
"c\n"

$ echo -e "a\nb\nc" | ruby -lne 'p $_'
"a"
"b"
"c"
```

It also causes `print` to behave like `puts`

```sh
$ ruby -e 'print "a"; print "b"'
ab

$ ruby -le 'print "a"; print "b"'
a
b
```


Exercise: Append line length to nonempty lines
----------------------------------------------

Here is an example input:

```sh
$ curl -sL 'https://raw.githubusercontent.com/ruby/ruby/trunk/BSDL'
```

Here is my solution:

```sh
$ curl -sL 'https://raw.githubusercontent.com/ruby/ruby/trunk/BSDL' | ruby -lne 'print $_, ~/./ && " (#{$_.length})"'
Copyright (C) 1993-2013 Yukihiro Matsumoto. All rights reserved. (64)

Redistribution and use in source and binary forms, with or without (66)
modification, are permitted provided that the following conditions (66)
are met: (8)
...
```


The `-a` flag splits the input and saves it in `$F`
---------------------------------------------------

```sh
$ echo -e "1 a\n22 bb\n333 ccc" | ruby -ane 'p $F'
["1", "a"]
["22", "bb"]
["333", "ccc"]
```


The `-F` flag sets the input field separator
--------------------------------------------

This will affect how the the `-a` flag splits the records.
Here, we'll use it to print the first three usernames from `/etc/passwd`

```sh
$ cat /etc/passwd | ruby -ne 'print unless /^#/' | ruby -F: -ane 'puts $F[0] if $. <= 3'
nobody
root
daemon
```


Dir globbing is a fast way to find files
----------------------------------------

The docs are [here](http://www.rubydoc.info/stdlib/core/Dir.glob),
Especially useful are single and double splats. Here, we use the
single splat to find all the C source files at the root directory of
Ruby, we then print the first 3

```sh
$ ruby -e 'puts Dir["*.c"].take 3'
addr2line.c
array.c
bignum.c
```


Backticks execute a system command and return its output
--------------------------------------------------------

```sh
$ ruby -e 'p `whoami`'
"josh\n"
```


Exercise: Find ruby files in the standard library
-------------------------------------------------

Output the first 5 files, alphabetically, that are anywhere in the standard library.

* You can find the stdlib's location with `gem which csv`.
* You can get the directory of a file with `File.dirname(filepath)`.
* You'll need to use the double splat for a recursive glob

Here is my solution:

```sh
$ ruby -e 'puts Dir["#{File.dirname `gem which csv`}/**/*.rb"].sort.take(5)'
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/English.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/abbrev.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/base64.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/benchmark.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/bigdecimal/jacobian.rb
```


You can change directories with `Dir.chdir`
-------------------------------------------

Also note that the block form will cd you back when done executing.

```sh
$ ruby -e 'p Dir.pwd; Dir.chdir("/"); p Dir.pwd'
"/Users/josh"
"/"

$ ruby -e 'p Dir.pwd; Dir.chdir("/") { p Dir.pwd}; p Dir.pwd'
"/Users/josh"
"/"
"/Users/josh"
```


Exercise: Same as before, but with relative paths
-------------------------------------------------

```sh
ruby -e 'Dir.chdir File.dirname `gem which csv`; puts Dir["**/*.rb"].sort.take(5)'
English.rb
abbrev.rb
base64.rb
benchmark.rb
bigdecimal/jacobian.rb
```


Exercise: Print the stdlib's 10 shortest files and their sizes
--------------------------------------------------------------

* It's much quicker to use `File.stat(filename).size` than `File.read(filename).size`,
  though they're not completely equivalent, but close enough for most uses.
* You'll probably want to use more than one Ruby program
* Note that `sort -n` will sort numerically instead of alphabetically

Here's my solution:

```sh
$ ruby -e 'puts Dir["#{File.dirname `gem which csv`}**/*.rb"]' \
>  | ruby -lne 'print File.stat($_).size, "\t", $_' \
>  | sort -n \
>  | head
50	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/drb.rb
59	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/optionparser.rb
85	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/tkfont.rb
85	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/tktext.rb
88	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/tkafter.rb
88	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/tkentry.rb
91	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/tkcanvas.rb
91	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/tkdialog.rb
91	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/tkmacpkg.rb
91	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/tkwinpkg.rb
```


Flipflops are useful in conditionals
------------------------------------

A flipflop looks like a range, but it's syntactically placed where a conditional is.
Its endpoints are boolean expressions. It evaluates to true when it's flipped on,
and `false` when it's flipped off.

It flips on when the LHS evaluates to true, and flips off when the RHS evaluates to true.

```sh
$ echo -e "a\nb\nc\nd\ne" | ruby -ne 'print if ($.==2)..($.==4)'
b
c
d
```

Ranges in conditionals are actually flip flops, they match against `$.`,
so we can simplify the above with:

```sh
echo -e "a\nb\nc\nd\ne" | ruby -ne 'print if 2..4'
b
c
d
```

Regexes as flipflop endpoints match against stdin.



Exercise: Print Ruby's description of `-n` from the manual
----------------------------------------------------------

* You can normalize the output of `man` by piping it through `col -b`
* Use a regexp flipflop, because it will be more resilient to change than line numbers

Here's my solution:

```
$ man ruby | col -b | ruby -ne 'print if /^\s*-n/../end/'
        -n      Causes Ruby to assume the following loop around your
                script, which makes it iterate over file name arguments
                somewhat like sed -n or awk.

                        while gets
                          ...
                        end
```


`gsub` is added to main and operates on `$_`
--------------------------------------------

There are several other methods like this, but `gsub` is the most useful one.

```sh
$ echo -e "cat\ncot\ncut" | ruby -pe 'gsub "c", "p"'
pat
pot
put
```



The `-n` and `-p` flags iterate over `ARGF`
-------------------------------------------

So it's not that these flags iterate over lines from stdin. Rather, they iterate
over lines from the program's input. It is `ARGF`'s job to figure out what that
means. If it sees no arguments in `ARGV`, then it will iterate over stdin. But
if it does see arguments, then it will assume they are filenames and iterate over
them as files.

This is useful with `-i` to do mass renamings in your project
(and git so you don't have to fear fucking it up).

Can also be useful with `$<.filename`


Exercise: Find the 10 most required files in the stdlib
-------------------------------------------------------

* Use `xargs` to place the input lines into `ARGV` for the Ruby script

```sh
$ ruby -e 'puts Dir["#{File.dirname `gem which csv`}/**/*.rb"]' \
>  | xargs ruby -ane 'puts $F[1] if /^\s*require /' \
>  | ruby -lpe $'gsub /["\']/, ""' \
>  | sort \
>  | uniq -c \
>  | sort -nr \
>  | head
 289 tk
  53 tkextlib/iwidgets.rb
  46 tkextlib/setup.rb
  35 tkextlib/tcllib.rb
  35 rubygems/command
  34 tkextlib/bwidget.rb
  27 rubygems
  26 thread
  26 fileutils
  24 tkextlib/blt.rb
```

`BEGIN`/`END` run before/after the script
-----------------------------------------

All `BEGIN` blocks will run before the first line, `END` will run before the program exits.
They are scoped to main, so you can use them to initialize variables.

Here, we'll use them to reverse a script.

```sh
$ echo -e "1\n2\n3" | ruby -ne '
>  BEGIN { lines = [] }
>  END   { puts lines.reverse }
>  lines << $_'
3
2
1
```

Exercise: Print the number of lines of input
--------------------------------------------

Here's my solution:

```sh
$ ruby -ne 'END { p $. }'
```


Exercise: Find the files in the stdlib with the most requires
-------------------------------------------------------------

```sh
$ ruby -e 'puts Dir["#{File.dirname `gem which csv`}/**/*.rb"]' | \
>  xargs ruby -r pp -ne $'
>  BEGIN { fns = Hash.new 0 }
>  fns[$<.filename] += 1 if /^\s*require /
>  END {
>    puts fns.sort_by(&:last).reverse.take(10).map { |f, n| "#{n}\t#{f}" }
>  }'
32	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/rubygems/resolver.rb
25	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/rubygems.rb
25	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/rubygems/test_case.rb
18	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/psych.rb
17	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/webrick.rb
14	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/rexml/document.rb
14	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/net/http.rb
14	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/rubygems/package.rb
13	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/rubygems/specification.rb
13	/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/rss/maker.rb
```


Exercise: Find the files in the stdlib that require `"date"`
------------------------------------------------------------

```sh
$ ruby -e 'puts Dir["#{File.dirname `gem which csv`}/**/*.rb"]' | \
>  xargs ruby -rpp -ane $'
>   BEGIN { fns = Hash.new { |h, k| h[k] = [] } }
>   fns[$F[1].gsub /[\'"]/, ""] << $<.filename if $F[0] == "require"
>   END { puts fns["date"] }
>  '
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/csv.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/json/add/date.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/json/add/date_time.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/optparse/date.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/psych/deprecated.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/psych/scalar_scanner.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/psych/visitors/to_ruby.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/time.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/xmlrpc/create.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/xmlrpc/datetime.rb
/Users/josh/.rubies/ruby-2.3.3/lib/ruby/2.3.0/xmlrpc/parser.rb
```


Exercise: Write head
--------------------

You can use `ruby -e 'puts *?a..?z'` to print the letters "a" through "z",
pipe them through a program that filters to the first 10 letters ("a" through "j")

```sh
# one solution
$ ruby -e 'puts *?a..?z' | ruby -ne 'print if 1..10'

# another solution
$ ruby -e 'puts *?a..?z' | ruby -ne 'print if $. <= 10'

# another solution
$ ruby -e 'puts *?a..?z' | ruby -pe 'exit if 10 < $.'
```


Exercise: Write tail
--------------------

Same as above, it should print "q" through "z".

```sh
$ ruby -e 'puts *?a..?z' | ruby -ne 'BEGIN { lines = [] }; lines << $_; END { puts lines.last 10 }'
```

Exercise: Write cat
-------------------

Replace `cat` in the script below with your program, it should have the same output.

```
cat `gem which json/version`
```

Here's my solution:

```sh
$ ruby -pe '' `gem which json/version`
```



Exercise: Find the gem with the longest name
--------------------------------------------

You can get the list of all gem names with `gem list --all --remote --no-versions`

Here's my solution:

```sh
$ gem list --all --remote --no-versions | ruby -e 'puts $stdin.readlines.sort_by { |g| -g.length }.take(10)'
ivyxxcspcqlaocvjbghawvbdartwsfffurhnqzlwvsbgieweawfntuwecdcminmiaunqteqgbrfuxppntjdvyvsswxwepnbfqstnrnsotrhndihkudyahthaxatviwrwtgllwbqhibouqctrxtypac
XHg4NFx4QzdceDE4cFx4QzRceEM5XHhGRVx4MDBceEVDXHg5Q1x4RUZceEI5XHhDMGlceEFFfkNOXGVcdlx4OTNceEE5XHhDNw
rabbit-slide-pmq20-tracing-a-memory-leak-in-a-long-running-eventmachine-application
xlmydsykwnrfbnvjffqcokoorkbskzzhrtgnzxkapmjtffjfkwcvwklmsrzwfiatwigrvmftpbybbeqi
rabbit-slide-kou-readable-code-workshop-for-pioneer-share-readable-code
multi_json_super_best_fun_with_streaming_amazing_its_a_spoon_not_a_fork
rabbit-slide-kou-mysql-and-postgresql-and-japanese-full-text-search-2
rabbit-slide-kou-mysql-and-postgresql-and-japanese-full-text-search-3
computer_please_do_you_happen_to_know_the_time_please_thank_you_meow
rabbit-slide-kou-mysql-and-postgresql-and-japanese-full-text-search
```

If you only needed the longest line, you could make this much more efficient by
tracking the longest one you'd seen so far, as you iterate over every line.
Give that one a try (use `BEGIN` and `END`):

```sh
$ cache gem list --all --remote --no-versions \
>  | ruby -lne '
>     BEGIN { longest = "" }
>     END   { puts longest }
>     longest = $_ if longest.length < $_.length
>    '
ivyxxcspcqlaocvjbghawvbdartwsfffurhnqzlwvsbgieweawfntuwecdcminmiaunqteqgbrfuxppntjdvyvsswxwepnbfqstnrnsotrhndihkudyahthaxatviwrwtgllwbqhibouqctrxtypac
```


Exercise: Display recent Ruby events
------------------------------------

You can get the list of events from https://api.github.com/repos/ruby/ruby/events

If you learn enough [jq](https://stedolan.github.io/jq/manual/),
this is where it shines, here's my `jq` solution:

```sh
$ curl -sL https://api.github.com/repos/ruby/ruby/events \
>  | jq -cr 'map([.created_at, .type, .actor.display_login]|join(" "))[]' \
>  | column -t
2017-05-28T16:29:53Z  PushEvent                      hsbt
2017-05-28T14:18:16Z  PushEvent                      hsbt
2017-05-28T13:27:47Z  CommitCommentEvent             MSP-Greg
...
```

But I've found it rather difficult to remember how it works right when I need it,
and I do know Ruby without having to go look anything up, so here's my Ruby solution:

```sh
$ curl -sL 'https://api.github.com/repos/ruby/ruby/events' \
>  | ruby -r json -e 'JSON.parse($stdin.read).map { |e| puts "#{e["created_at"]} #{e["type"]} #{e["actor"]["display_login"]}" }' \
>  | column -t
2017-05-28T16:29:53Z  PushEvent                      hsbt
2017-05-28T14:18:16Z  PushEvent                      hsbt
2017-05-28T13:27:47Z  CommitCommentEvent             MSP-Greg
...
```
