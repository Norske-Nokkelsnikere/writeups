CTF: [LIT CTF 2023](https://ctftime.org/event/2052) 
Category: Reverse
Description: I love regex.
Challenge points: 41 solves / 213 points
Writeup date: 2023-08-08

## Where to start

The challenges has provided a file that is called regex.txt
Here we get to see this regex:
```^LITCTF\{(?<=(?=.{42}(?!.)).(?=.{24}greg).(?=.{30}gex).{5})(?=.{4}(.).{19}\1)(?=.{4}(.).{18}\2)(?=.{6}(.).{2}\3)(?=.{3}(.).{11}\4)(?=.{3}(.).{3}\5)(?=.{16}(.).{4}\6)(?=.{27}(.).{4}\7)(?=.{12}(.).{4}\8)(?=.{3}(.).{8}\9)(?=.{18}(.).{2}\10)(?=.{4}(.).{20}\11)(?=.{11}(.).{2}\12)(?=.{32}(.).{0}\13)(?=.{3}(.).{24}\14)(?=.{12}(.).{9}\15)(?=.{7}(.).{2}\16)(?=.{0}(.).{12}\17)(?=.{13}(.).{5}\18)(?=.{1}(.).{0}\19)(?=.{27}(.).{3}\20)(?=.{8}(.).{17}\21)(?=.{16}(.).{6}\22)(?=.{6}(.).{6}\23)(?=.{0}(.).{1}\24)(?=.{8}(.).{11}\25)(?=.{5}(.).{16}\26)(?=.{29}(.).{1}\27)(?=.{4}(.).{9}\28)(?=.{5}(.).{24}\29)(?=.{15}(.).{10}\30).*}$```

This looks like a cat has played on the keyboard, and pressing a lot of random buttons.
![[how-to-regex.png]]

So, where do we start? A teammate were using dcode to analyze the regex. I've used regexp before, but noticed it didn't like first part of the regex because of the positive lookbehind. [Dcode regex analyzer](https://www.dcode.fr/regular-expression-analyser) had no problems with this regex.

Dcode website creates a useful railroad diagram that better shows how it works.
![[regex-railroad-diagram.png]]

Still not giving me a lot of clues except that it has string greg and gex, which can be used to spell regex! I decided to take a little break doing other challenges before I went to do this again. 

## Starting to reverse.

## Part 1 - Greg and Gex
I started with taking the first part that has greg and gex. 
``LITCTF\{(?<=(?=.{42}(?!.)).(?=.{24}greg).(?=.{30}gex).{5})``
Since I don't really use regex a lot, I haven't learned the advanced regex that is used here.
So I learned that (?<=) is positive lookbehind, (?!) is negative lookahead, (?=) is positive lookahead. I used https://regex101.com/ to mess around with it to understand it.


``(?=.{42}(?!.))`` I found out that this tells regex to match from index 0 if there are 42 characters before the match. So we know the length of string is 42 characters.
``.(?=.{24}greg)`` tells us to match any character 25 times and then greg.
``.(?=.{30}gex)`` tells us to match any character 31 times and then gex. 
``.{5}`` tells us that we match any characters for the last 7 (5 starting from 0)
So with this knowledge we know that 
``LITCTF\{(?<=(?=.{42}(?!.)).(?=.{24}greg).(?=.{30}gex).{5})`` will match the string ``LITCTF{..................greg...gex.......`` 
I used a full stop is for characters we don't know.

## Part 2 - Matching characters

```
(?=.{4}(.).{19}\1)
(?=.{4}(.).{18}\2)
(?=.{6}(.).{2}\3)
(?=.{3}(.).{11}\4)
(?=.{3}(.).{3}\5)
(?=.{16}(.).{4}\6)
(?=.{27}(.).{4}\7)
(?=.{12}(.).{4}\8)
.....
```
If we format the regex in a better way then we can notice a pattern.
We see the ?= positive lookahead and ``\x`` where x increases until it's 30. So we have 30 of these regex conditions. 

Lets cut it down into more pieces
```
(?=
.{4}
(.)
.{19}
\1
)
```
We are first doing a positive lookahead, where we capture the 5th character into a capture group. (.) tells us that we should capture any characters and then we match 20 that can be any characters before we match the capture group.

A better example would be ``(?=.{0}(.).{2}\1)`` 
For it to match we need to have a string where first character and third after the first character are the same. 1001 would work, but 1110 wouldn't.

So I started doing adding each pattern up and matching them. Most efficient would be writing a script that does it for you, but I decided to waste some time on this and did it manually. It took a 3 times before I was able to do it (because human error).

In the end I got the string that matched with the regex
``LITCTF{rrregereeregergegegregegggexexexxx}``






