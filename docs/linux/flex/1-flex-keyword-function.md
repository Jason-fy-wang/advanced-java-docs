---
tags:
  - flex
  - keyword
---
flex中的一些功能记录.

##### 1. structural
```flex
Definitions section
%%
Rules definitions
%%
user code section



```

```c
Definitions section:
name definition: 
digit [0-9]

%option : define some character of scanner.  (noyywrap)
%s: define inclusive state. if state is inclusive, then rules with no states at all will also be active. 
%x: define exclusive state if state is exclusive, the only rules qualified with the start condition will be active. 

%top {
// top code goes to the top of generated file
#include<stdio.h>
}

```

```flex
options:
noyywrap: yylex() receives an end-of-file indication, it invokes yywrap(). if it returns false(zero), then yylex() resumes that yyin(the global input file) has been redirected to another source and scanning continues; if it returns true(non-zero), then the scanning process terminates.  Option noyywrap make yywrap() to return always true.

```

```flex
states:

examples:
%s example
%%
<example>foo do_something();
bat somethnig_else()


equivalent to:
%x example
%%
<example>foo do_something();
<INITIAL,example>bar something_else();

# using BEGIN action to switch to next state
%x comment 
%%
"/*"  {BEGIN(commend);}
<comment>.  {;}
<comment>\n|\r|\r\n  {;}
<comment>"*/" {BEGIN(INITIAL);}


```

```flex
Rules definitions

pattern action:
({id})   {printf("path correct: %s\n", yytext)}


```

```flex
user code definitions

This section is simply copied to lex.yy.c verbatim.


```

##### 2. keyword
```flex

yytext:  points to the first character of the match in the input buffer.


```



