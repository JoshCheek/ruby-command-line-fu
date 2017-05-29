Ruby Command Line Fu - Replacing sed
====================================

I searched for "sed examples" and found [this](http://sed.sourceforge.net/sed1line.txt)
list. I've rewritten most of the solutions in Ruby.

File Spacing:

* Double space a file `ruby -ne 'puts $_, ""'`
* Double space a file which already has blank lines in it.
  Output file should contain no more than one blank line between lines of text.
  `ruby -ne 'puts $_, "" unless /^\s*$/'`
* Triple space a file `ruby -ne 'puts $_, "", ""'`
* Undo double-spacing (assumes even-numbered lines are always blank)
  `ruby -ne 'print if $..odd?'`
* Insert a blank line above every line which matches "regex"
  `ruby -pe 'puts if /regex/'`
* Insert a blank line below every line which matches "regex"
  `ruby -pe '$_ << "\n" if /regex/'`
* Insert a blank line above and below every line which matches "regex"
  `ruby -pe '$_ = "\n#$_\n" if /regex/'`

Numbering:

* Number each line of a file (simple left alignment).
  Using a tab instead of space will preserve margins.
  `ruby -ne 'print "#$.\t#$_"'`
* Number each line of a file (number on left, right-aligned)
  `ruby -ne 'printf "%6d  %s", $., $_'`
  Note: my output here is set up to match their example sed script's.
* Number each line of file, but only print numbers if line is not blank
  `ruby -pe 'print "#$. " unless /^\s*$/'`
* Count lines (emulates "wc -l") `ruby -ne 'END { p $. }'`

Text Conversion and Substitution:

* IN UNIX ENVIRONMENT: convert DOS newlines (CR/LF) to Unix format.
  `ruby -lpe ''`
* IN UNIX ENVIRONMENT: convert Unix newlines (LF) to DOS format.
  `ruby -pe '$_ = $_.chomp << "\r\n"'`
* IN DOS ENVIRONMENT: convert Unix newlines (LF) to DOS format.
  Uhhhh... same as above.
* Delete leading whitespace (spaces, tabs) from front of each line.
  Aligns all text flush left.
  `ruby -pe 'sub /^\s*/, ""'`
* Delete trailing whitespace (spaces, tabs) from end of each line.
  `ruby -pe '$_.rstrip!'`
* Delete BOTH leading and trailing whitespace from each line.
  `ruby -pe '$_.strip!'`
* Insert 5 blank spaces at beginning of each line (make page offset)
  `ruby -pe 'print " "*5'`
* Align all text flush right on a 79-column width
  `ruby -lne 'puts $_.rjust(79)'`
* Center all text in the middle of 79-column width.
  `ruby -lne 'puts $_.center(79)'`
* Substitute (find and replace) "foo" with "bar" on each line.
  * Only the first instance `ruby -pe 'sub "foo", "bar"'`
  * Every instance `ruby -pe 'gsub "foo", "bar"'`
* Substitute "foo" with "bar" ONLY for lines which contain "baz"
  `ruby -pe 'gsub "foo", "bar" if /baz/'`
* Substitute "foo" with "bar" EXCEPT for lines which contain "baz"
  `ruby -pe 'gsub "foo", "bar" unless /baz/'`
* Change "scarlet" or "ruby" or "puce" to "red"
  `ruby -pe 'gsub "scarlet", "puce"; gsub "ruby", "red"'`
* Reverse order of lines (emulates "tac")
  `ruby -ne 'BEGIN { lines = [] }; lines << $_; END { puts lines.reverse }'`
* Reverse each character on the line (emulates "rev")
  `ruby -pe '$_.reverse!'`
* Join pairs of lines side-by-side (like "paste")
  `ruby -ne '$..odd? ? print($_.chomp) : print(" ", $_)'`
* If a line ends with a backslash, append the next line to it
  `ruby -pe '$_.chomp! "\\\\\n"'`
* If a line begins with an equal sign, append it to the previous line
  and replace the "=" with a single space.
  `ruby -075 -pe 'sub /\n=\z/, " "'`
  This one has ruby split the lines by equal signs (octal number 75) instead of newlines.
* Add commas to numeric strings, changing "1234567" to "1,234,567"
  `ruby -pe 'gsub(/\d+/) { |n| n.reverse.scan(/\d{1,3}/).join(",").reverse }'`
* Add commas to numbers with decimal points and minus signs
  `ruby -pe 'def commas(s) s.scan(/.{1,3}/).join(",") end; gsub(/(\d+)(\.)?(\d+)?/) { |n| commas($1.reverse).reverse + $2.to_s + commas($3.to_s) }'`
* Add a blank line every 5 lines (after lines 5, 10, 15, 20, etc.)
  `ruby -ne 'print; puts if $. % 5 == 0'`

Selective Printing of Certain Lines:

* Print first 10 lines of file (emulates behavior of "head")
  `ruby -ne 'print if 1..10'`
* Print first line of file (emulates "head -1")
  `ruby -ne 'print if $. == 1'`
* Print the last 10 lines of a file (emulates "tail")
  `ruby -ne 'BEGIN { lines = [] }; END { puts lines }; lines << $_; lines.shift if lines.length > 10'`
* Print the last 2 lines of a file (emulates "tail -2")
  `ruby -ne 'BEGIN { lines = [] }; END { puts lines }; lines << $_; lines.shift if lines.length > 2'`
* Print the last line of a file (emulates "tail -1")
  `ruby -ne 'l = $_; END { print l }'`
* Print the next-to-the-last line of a file
  `ruby -ne 'l = l.to_a[1], $_; END { print l[0] }'`
  This one's a little bit tricky, there are less clever ways to do it, but they're longer.
* Print only lines which match regular expression (emulates "grep")
  `ruby -ne 'print if /regexp/'`
* Print only lines which do NOT match regexp (emulates "grep -v")
  `ruby -ne 'print unless /regexp/'`
* Print the line immediately before a regexp, but not the line containing the regexp
  `ruby -ne 'BEGIN { prev = "" }; print prev if /true/; prev = $_'`
* Print the line immediately after a regexp, but not the line containing the regexp
  `ruby -ne 'BEGIN { pprev = false }; print if pprev; pprev = ~/regexp/'`
* Print 1 line of context before and after regexp, with line number indicating where
  the regexp occurred (similar to "grep -A1 -B1")
  `ruby -ne 'BEGIN { lines = []; tp = {} }; END { lines.each.with_index(1) { |l, i| print l if tp[i]; puts "---" if tp[i]&&!tp[i+1] } }; lines << $_; tp[$.-1] = tp[$.] = tp[$.+1] = true if /regexp/'`
* Grep for AAA and BBB and CCC (in any order)
  `ruby -ne 'print if /AAA/ && /BBB/ && /CCC/'`
* Grep for AAA and BBB and CCC (in that order)
  `ruby -ne 'print if /AAA.*BBB.*CCC/'`
* Grep for AAA or BBB or CCC (emulates "egrep")
  `ruby -ne 'print if /AAA/ || /BBB/ || /CCC/'`
* Print paragraph if it contains AAA (blank lines separate paragraphs)
  `ruby -00ne 'print if /AAA/'`
* Print paragraph if it contains AAA and BBB and CCC (in any order)
  `ruby -00ne 'print if /AAA/' && /BBB/ && /CCC/`
* Print paragraph if it contains AAA or BBB or CCC
  `ruby -00ne 'print if /AAA/' || /BBB/ || /CCC/`
* Print only lines of 65 characters or longer
  `ruby -lne 'print if $_.length >= 65'`
* Print only lines of less than 65 characters
  `ruby -lne 'print if $_.length < 65'`
* Print section of file from regular expression to end of file
  `ruby -ne $'print $\' if /regex/'`
  This one uses the Perl style global variable `$'`,
  which holds the string after the match.
* Print section of file based on line numbers (lines 8-12, inclusive)
  `ruby -ne 'print if 8..12'`
* Print line number 52
  `ruby -ne 'print if $. == 52'`
* Beginning at line 3, print every 7th line
  `ruby -ne 'print if $. >= 3 && ($.-3) % 7 == 0'`
* Print section of file between two regular expressions (inclusive)
  `ruby -ne 'print if /Iowa/../Montana/'`

Selective Deletion of Certain Lines:

* Print all of file EXCEPT section between 2 regular expressions
  `ruby -ne print unless /Iowa/../Montana/`
* Delete duplicate, consecutive lines from a file (emulates "uniq").
  First line in a set of duplicate lines is kept, rest are deleted.
  `ruby -ne 'print if $. == 1 || $_ != @prev; @prev = $_'`
* Delete duplicate, nonconsecutive lines from a file.
  `ruby -ne 'BEGIN { lines = {} }; print unless lines[$_]; lines[$_] = true'`
* Delete all lines except duplicate lines (emulates "uniq -d").
  `ruby -ne 'BEGIN { lines = {} }; print if lines[$_]; lines[$_] = true'`
* Delete the first 10 lines of a file
  `ruby -ne 'print unless 1..10'`
* Delete the last line of a file
  `ruby -ne 'print if $<.eof?'`
  I just realized this was probably a thing, and it thankfully was!
* Delete the last 2 lines of a file
  `ruby -ne 'BEGIN { lines = [] }; lines << $_; print lines.shift unless lines.length <= 2'`
* Delete the last 10 lines of a file
  `ruby -ne 'BEGIN { lines = [] }; lines << $_; print lines.shift unless lines.length <= 10'`
* Delete every 8th line
  `ruby -ne 'print unless $. % 8 == 0'`
* Delete lines matching pattern
  `ruby -ne 'print unless /pattern/'`
* Delete ALL blank lines from a file (same as "grep '.' ")
  `ruby -ne 'print unless /^$/'`
* Delete all CONSECUTIVE blank lines from file except the first 2:
  `ruby -e 'print $stdin.read.gsub(/\n{2,}/m).with_index { |nls, i| nls if i.zero? }'`
* Delete all leading blank lines at top of file
  `ruby -ne 'print if @on ||= ~/./'`
* Delete all trailing blank lines at end of file
  `ruby -ne 'BEGIN { nls = [] }; next nls << $_ if /^$/; puts nls; nls = []; print'`
