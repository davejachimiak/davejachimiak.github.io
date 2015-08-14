---
layout: post-no-feature
title: "A Lispy Calculator with Bison"
date: 2015-08-14
categories: parsing interpreters
---

Bison is a parser generator that's backwards compatable with
[yacc](http://dinosaur.compilertools.net/yacc/). You give it a lexer and
a grammar in a .y file, and it gives you a parser.

Bison's manual includes [examples of grammars for different kinds of
calculators](https://www.gnu.org/software/bison/manual/html_node/Examples.html#Examples).
The Lispy calculator below is based on those examples.

The manual's calculators are REPLs if you run them without sending
anything to stdin. They print the values of valid expressions when the
user presses return.

Using a Lispy calculator would look like this in your terminal:

{% highlight lisp %}
$ ./lispcalc
10
;Value: 10
(+ 22 20)
;Value: 42
(- 100 50)
;Value: 50
(* (+ 8 2) (/ 20 2))
;Value: 100
(- 10)
syntax error
{% endhighlight %}

The grammar rules section for the .y file looks like this.

{% highlight c %}
input
: %empty
| input line
;

line
: '\n'
| expression '\n' { printf (";Value: %.10g\n", $1); }
;

expression
: NUM { $$ = $1; }
| '(' '+' expression expression ')' { $$ = $3 + $4; }
| '(' '-' expression expression ')' { $$ = $3 - $4; }
| '(' '*' expression expression ')' { $$ = $3 * $4; }
| '(' '/' expression expression ')' { $$ = $3 / $4; }
;
{% endhighlight %}

The simplicity of this grammar is due to Lisp's prefix notation.
Operator precedence is not a concern. This is unlike grammars with infix
notation, where precedence makes parser generators and the automata they
create more complex.

The parser gets lookahead tokens by invoking a lexer contained in the
`yylex` function. Lexers can built with
[lex](http://dinosaur.compilertools.net/lex/), which exports the `yylex`
function for yacc-like generators. `yylex` can also be designed by hand.
Our calculator can use the [Bison manual's lexer for its reverse polish
notation
calculator](https://www.gnu.org/software/bison/manual/html_node/Rpcalc-Lexer.html#Rpcalc-Lexer).

{% highlight c %}
#include <ctype.h>

int
yylex (void)
{
  int c;

  /* Skip white space.  */
  while ((c = getchar ()) == ' ' || c == '\t')
    continue;
  /* Process numbers.  */
  if (c == '.' || isdigit (c))
    {
      ungetc (c, stdin);
      scanf ("%lf", &yylval);
      return NUM;
    }
  /* Return end-of-input.  */
  if (c == EOF)
    return 0;
  /* Return a single char.  */
  return c;
}
{% endhighlight %}

The complete lispcalc.y file can be found
[here](https://gist.github.com/davejachimiak/cf406cf8886077161a76). That
gist includes a Makefile that assumes version 3.0.4 of Bison installed
via Homebrew on OSX.

<hr>

Discussing parser generators is unsatisfying when not talking about
exactly what they generate. For example, Bison's default is to generate
bottom-up, LALR(1) parsers that use a shift-reduce algorithm. But I'll
talk about all of that in other blog posts.
