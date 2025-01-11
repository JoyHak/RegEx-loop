### Regex Invention
![](https://github.com/JoyHak/RegEx-loop/blob/main/Video_2025_01_11-1.mp4)
I want to share my invention, which allows you to use only one RegEx, which will search for substrings in a string with a certain condition. If you want, call it a RegEx loop that didn't exist before!

I will share with you not only options for implementing loops in RegEx, but also show various examples from simple to complex.

### Problem

For the [EnhanceAnyLexer](https://github.com/Ekopalypse/EnhanceAnyLexer) Notepad++ plugin, I needed to write a RegEx in one line that would look for substrings in a string. Due to the limitations of the plugin, I could only use RegEx. My task was to highlight individual words only inside quotation marks and avoid certain characters. It is impossible to use **look ahead/behind** *(the words are far from the quotes)*, the conditions `(?(condition)|(true)(false))` and subgroups with a quantifier `( )+` because *a repeated capturing group will only capture the last iteration*. 

I have not found a clean RegEx on the Internet that would perform this task. [This expression is the closest solution to my problem.](https://stackoverflow.com/a/51667506)

Based on this answer, I came up with different versions of RegEx that cyclically search for expressions based on a certain condition. The [Perl syntax](https://www.boost.org/doc/libs/1_85_0/libs/regex/doc/html/boost_regex/syntax/perl_syntax.html) is used here. 

#### Explanation

Let's start with a simple task. If there is a `c` at the beginning of the line highlight only words and numbers in it:

```tcl
c =word &word 2-2=0 
c 22 ^&*w0rd* 
+word &word   
```

We expect to highlight only the words `word` and the numbers `2` and `0` only in the first two lines. The template for a similar task looks like this:

```python
# Condition under which the second part will be triggered
(?: 
    condition \K  # Match the condition and reset the match start
)

|  			# Beginning of the loop

(?<=\G)  		# Ensures that the condition is found; the loop starts from the position of the condition/previous iteration
separator*?  	# Ungreedy: zero and more separators before expresseion: space, char, ...
\K  			# Reset the match start again
expression  	# Match the expression: \w+ or .+ ...
```

`\K` means *forget everything before and start highlight from this position*. It helps not to highlight everything that was before `\K`.  Now we construct the RegEx according to the template ([DEMO](https://regex101.com/r/wXPPD2/1)):

```python
(?: 
    c \K  		# Condition: "c" literal
)

|  			# Beginning of the loop

(?<=\G)  		# Ensures that the condition is found; the loop starts from the position of the condition/previous iteration
.*?  			# Ungreedy separator: zero and more chars/spaces
\K  			# Reset the match start again
\w+  			# Greedy: matches any words/numbers (equivalent to [a-zA-Z0-9_])
```

However, such a RegEx highlights the words **to the end of the entire line**: it does not have a `stop condition`. Let's complicate the same task: if there is a `c` at the beginning of the line and then any quotes `" '` highlight only words and numbers in quotes:

```tcl
c "=word &word" 2-2=0 
c "22" ^&*w0rd* 
+word &word   
```

The final template looks like this: 

```python
(?: 
    condition\K  # Match the condition and reset the match start
)

|  			# Beginning of the loop

(?<=\G)  		# Ensures that the condition is found
stop*?  		# The character at which the entire regex stops: [^exclude]
\K  			# Reset the match start again
expression  	# Match the expression: \w+ or .+ ...
```

Now we construct the RegEx according to the template. The template is similar to the previous one, but quotes `[" ']` and space/tab `[ \t]` are added to the condition: `c[ \t]*?["']` The RegEx stop condition appears: `[^"']`*(everything except the quotes.)*. After it, the highlighting of further characters is completed. Now we construct the RegEx according to the template ([DEMO](https://regex101.com/r/FUH7Xx/1)):

```python
(?: 
    c[ \t]*?["']\K  		# Condition: c " or c ' or c   " or c   '
)

|  			# Beginning of the loop

(?<=\G)  		# Ensures that the condition is found
[^"']*?  		# Quotes after which the entire regex stops
\K  			# Reset the match start again
\w+  			# Greedy: matches any words/numbers 
```

Try to solve these problems without using these templates. I will be very glad if you find the optimal solution!

#### Final template

Find a quoted string and highlight only chars that are not in brackets `{ }` : 

```tcl
"= {&+%} %" ... 
"{&+%}=={&+%}$" ...
{!} word and {$} anything else  
```

Simply put, we have to walk along the line, bypassing everything that is enclosed in `{ }` . In this case, there should be two different conditions for stopping: the condition for `stopping and repeating` the cycle if parentheses are found `{ }`; the `condition for stopping` if quotes are found `" '`:

```python
# Condition under which the second part will be triggered
^condition

|  		# Beginning of the loop

(?<=\G)  	# Ensures that the condition is found; the loop starts from the position of the condition/previous iteration

(?>      	# Start of the atomic group
    skip 		# content to skip: {.*}
 	|  			# OR
    stop    	# The character at which the entire regex stops: [^exclude]
) \K      	# Resets the match, so it is not included in the output

expression  # Match the expression: \w+ or .+ ...
```

The idea: after encountering a `condition`, the RegEx starts from its position `(?<=\G)` goes on, stops if a quotation mark is detected, bypasses the group, resets the current position and finally captures the desired `expression`. Having reached the end, all the RegEx is repeated again from the position of the last found word `(?<=\G)`. And so on until the main stop condition is met.

The atomic group `(?>...)` is used to group sub-patterns that should not be subject to backtracking. This means that once the group finds a match, it is fixed, and the regex engine will not attempt to change this match, even if subsequent parts of the pattern do not match. In this context, the atomic group allows for efficient handling of nested structures, such as curly braces, and ensures that if a match is found within the curly braces, it will not be reconsidered.

The final version ([DEMO 1,](https://regex101.com/r/vZugRo/1) [DEMO 2](https://regex101.com/r/uKMPHi/1)):

```py
^["']\K # Matches a starting quote (single or double) and discards it from the output

|  		# Beginning of the loop

(?>      	# Start of the atomic group
    {.*?}   	# Skips the content within curly braces { }
 	|			# OR
    [^"']    	# Stops RegEx if quotes are found
) \K      	# Resets the match, so it is not included in the output

[^{}"']+  	# Matches one or more characters that are not {, }, ", or '
```

#### Limitations

I will be very glad if you find another optimal solution! Please help me improve these templates. They have significant optimization problems: if `condition` is not found, `alternation (?<=\G) is checked for each character`; there is no skipping of unsuitable lines; flags do not work `(*SKIP)(*F)`. Despite the fast speed of work, the number of steps tends to 100,000.

I will wait for the plugin to improve so that I don't have to solve such problems.…
