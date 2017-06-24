# Parsing Preprocessor Directives in Objective-C Code

The programming language with uses [preprocessor directives](https://en.wikipedia.org/wiki/Directive_(programming)) is difficult to parse, since it requires calculating the values for the
directives, pulling out uncompilable code fragments, and parsing of the
clean code. Parsing of directives and the normal code also can be
performed simultaneously. This article goes into detail describing both
approaches to Objective-C, revealing their strengths and weaknesses.
Besides the theoretical side of the question, these approaches are
practically implemented in web services, such as Swiftify and Codebeat.

[<img align="left" src="https://habrastorage.org/files/d3c/d53/8db/d3cd538db7604fe3ad10f759a9042d76.jpg"/>](https://objectivec2swift.com/)
[**Swiftify**](https://objectivec2swift.com/) is a web application for
converting the Objective-C source code to Swift. Currently, the service
supports processing of both single files and entire projects. This
service can reduce the time spent by developers who want to learn a new
programming language from Apple.

<br>
<br>

[<img align="left" src="https://habrastorage.org/files/f81/032/e83/f81032e83f4d45cba5a529fad9df9834.png"/>](https://codebeat.co/)
[**Codebeat**](https://codebeat.co/) is an automated system for
calculating code metrics and analyzing different programming languages,
including Objective-C.

<!-- cut -->
<br>
<br>
<br>

## Contents

* [Introduction](#introduction)
* [One-step processing](#one-step-processing)
    * [Associating Hidden Tokens with Non-terminal Nodes](#associating-hidden-tokens-with-non-terminal-nodes)
    * [Associating Hidden Tokens with Terminal Nodes](#associating-hidden-tokens-with-terminal-nodes)
    * [Ignored macros](#ignored-macros)
* [Two-step Processing](#two-step-processing)
    * [Preprocessing Lexer](#preprocessing-lexer)
    * [Preprocessing Parser](#preprocessing-parser)
    * [Preprocessor](#preprocessor)
    * [Lexer](#lexer)
    * [Parser](#parser)
* [Other Processing Approaches](#other-processing-approaches)
* [Conclusion](#conclusion)

## Introduction

Processing of preprocessor directives is performed during the parsing of
the normal code. Although I am not going into detail with basic notions
of parsing,
some [terms](http://blog.ptsecurity.com/2016/06/theory-and-practice-of-source-code.html) from
the article on the source code parsing with ANTLR and Roslyn will
appear. Both services use ANTLR as a parser generator. Objective-C
grammars are available in official ANTLR grammars repository.
([Objective-C grammar](https://github.com/antlr/grammars-v4/tree/master/objc)).

We distinguished two ways of parsing preprocessor directives:

* One-step Processing
* Two-step Processing

## One-step Processing

One-step processing involves the simultaneous parsing of directives and
tokens of the primary language. ANTLR introduces a system of channels
isolating tokens by their type. For example, tokens of the primary
language and hidden tokens (whitespaces and comments). The directive
tokens can be added to a separate named channel.

The directive tokens normally start with a `#` character (or sharp) and
end with newline characters (`\r\n`). Therefore, it is advisable to use
a different lexical mode if you need to capture such tokens. ANTLR
supports lexical modes, which can be described as follows: `mode DIRECTIVE_MODE;`. Below is the fragment of the lexer containing the mode
section for preprocessor directives:

```ANTLR
SHARP:  '#'                    -> channel(DIRECTIVE_CHANNEL), mode(DIRECTIVE_MODE);

mode DIRECTIVE_MODE;

DIRECTIVE_IMPORT:              'import' [ \t]+  -> channel(DIRECTIVE_CHANNEL), mode(DIRECTIVE_TEXT_MODE);
DIRECTIVE_INCLUDE:             'include' [ \t]+ -> channel(DIRECTIVE_CHANNEL), mode(DIRECTIVE_TEXT_MODE);
DIRECTIVE_PRAGMA:              'pragma'         -> channel(DIRECTIVE_CHANNEL), mode(DIRECTIVE_TEXT_MODE);
```

Part of the Objective-C preprocessor directives can be converted into a
particular Swift code (for example, using the **let** syntax). Some
directives stay the same while the rest are converted to comments. The
table below gives some examples.

| Objective-C | Swift  |
|---|---|
| `#define SERVICE_UUID        @ "c381de0d-32bb-8224-c540-e8ba9a620152"`  | `let SERVICE_UUID = "c381de0d-32bb-8224-c540-e8ba9a620152"`  |
| `#define ApplicationDelegate ((AppDelegate *)[UIApplication sharedApplication].delegate)`  | `let ApplicationDelegate = (UIApplication.shared.delegate as? AppDelegate)`  |
| `#define DEGREES_TO_RADIANS(degrees) (M_PI * (degrees) / 180)`  | `func DEGREES_TO_RADIANS(degrees: Double) -> Double { return (.pi * degrees)/180; }`  |
| `#if defined(__IPHONE_OS_VERSION_MIN_REQUIRED)`  | `#if __IPHONE_OS_VERSION_MIN_REQUIRED`  |
| `#pragma mark - Directive between comments.` | `// MARK: - Directive between comments.`

Comments in the resulting Swift code should be placed in the correct
position. However, as already mentioned, the parse tree does not contain
hidden tokens themselves.

### What if we add hidden tokens to parse tree?

Actually, hidden tokens can be included in the grammar; though, it will make 
grammar too complex and redundant (as every rule will contain Comment and Directive 
tokens placed between the significant tokens).

```ANTLR
declaration: property COMMENT* COLON COMMENT* expr COMMENT* prio?;
```

Therefore, this approach is no good at all.

However, how can we retrieve these tokens when traversing the parse tree?

It turned out that there are several ways to solve this problem allowing
you to associate hidden tokens with non-terminal or terminal nodes
of the parse tree.

### Associating Hidden Tokens with Non-terminal Nodes

This method was derived from a relatively old article of [2012 on ANTLR 3](http://meri-stuff.blogspot.ru/2012/09/tackling-comments-in-antlr-compiler.html).

In this case, all hidden tokens are classified into the following types:

* Preceding
* Following
* Orphans

To understand the nuances of these types, let's consider a rule where
curly brackets are terminal characters and any expression ending with a
semicolon can be used as a `statement`  (for example, assignment `a = b;`).

```ANTLR
root
    : '{' statement* '}'
    ;
```

In this case, all the comments from the following code fragment will be
placed on the preceding list. (The first token in a file or tokens
before the non-terminal nodes of the parse tree.)

```Java
/*First comment*/ '{' /*Precending1*/ a = b; /*Precending2*/ b = c; '}'
```

If a comment is the last in the file, or it is inserted after
all `statement` (there is a terminal bracket after it), this comment will
be placed on the following list.

```Java
'{' a = b; b = c; /*Following*/ '}' /*Last comment*/ 
```

All other comments are listed as orphans. (In fact, they are separated
by tokens or by curly brackets in this context.):

```Java
'{' /*Orphan*/ '}'
```

This decomposition approach allows for processing of hidden tokens in
the common `Visit` method. The above-mentioned approach is used in
Swiftify, but it could hardly be used to generate a **full-fidelity**
parse tree. Fidelity means that the parse tree can be converted back
into the same character-for-character code, including spaces, comments,
and preprocessor directives. Afterward we are going to use the
technique for handling preprocessor directives as described below.

### Associating Hidden Tokens with Terminal Nodes

In this case, hidden tokens are associated with significant ones.
A hidden token can be either **leading** or **trailing**. This technique is currently used in
Roslyn parser (for C# and Visual Basic). According to this approach,
hidden tokens are called Trivia.

A set of trailing trivia tokens includes all trivia in the same string
from one significant token to another. All of the other hidden tokens
fall within the set of leading tokens and become associated with the
subsequent significant token. The first significant token contains the
initial trivia. Hidden tokens at the end of the file are associated with
the latter special end-of-file token of zero length. For more
information about the types of parse tree structures and Trivia, see the
official documentation
on [Roslyn](https://github.com/dotnet/roslyn/wiki/Roslyn%20Overview).

There is a method in ANTLR for the token with index i, which returns all
tokens on the specified channel to the left or to the
right: `getHiddenTokensToLeft(int tokenIndex, int channel)`, `getHiddenTokensToRight(int tokenIndex, int channel)`. Thus, we
can make the ANTLR-based parser build a full-fidelity parse tree similar
to a Roslyn parse tree.

### Ignored Macros

As macros are not replaced by the Objective-C code fragments during the
one-step processing, they can be ignored or placed in a separate
channel. This helps to avoid problems with parsing normal Objective-C
code and the need to include macros in grammar nodes (by analogy with
the comments). This also applies to the default macros such
as `NS_ASSUME_NONNULL_BEGIN`, `NS_AVAILABLE_IOS(3_0)`, and others:

```ANTLR
NS_ASSUME_NONNULL_BEGIN : 'NS_ASSUME_NONNULL_BEGIN' ~[\r\n]*  -> channel(IGNORED_MACROS);
IOS_SUFFIX              : [_A-Z]+ '_IOS(' ~')'+ ')'           -> channel(IGNORED_MACROS);
```

## Two-step Processing

Two-step processing algorithm can be represented as the following
sequence of steps:

1.  Tokenization and parsing of code for preprocessor directives. At
    this step, common code fragments are recognized as plain text
2.  Determining the conditional directives (`#if`, `#elif`, and `#else`)
    and compiled code blocks
3.  Calculating and applying values of the `#define` directives to the
    appropriate place in the compiled code blocks
4.  Replacing directives from the source code by space characters (to
    save the correct position of tokens in the source code)
5.  Tokenization and parsing of the target text from which directives
    were removed

However, you can omit the third step and include macros directly in the
grammar, at least partially. This technique is more difficult to
implement comparing to one-step processing approach: if there is a need
to maintain the correct position of tokens in the normal source code, it
requires replacing the code for preprocessor directives with the space
characters. However, this algorithm for handling preprocessor directives
was practically implemented in Codebeat. Grammars and a visitor for
processing preprocessor directives are available
on [GitHub](https://github.com/antlr/grammars-v4/tree/master/objc/two-step-processing).
An additional advantage of this approach is the representation of
grammars in a more structured manner.

The following components are used for the two-step processing:

1. Preprocessing lexer
2. Preprocessing parser
3. Preprocessor
4. Lexer
5. Parser

I would like to remind that the **lexer** converts source code
characters into significant elements called lexemes or tokens. The 
**parser** recognizes a stream of tokens into the tree structure,
which is called a parse tree. **Visitor** is a design pattern that
allows you to encapsulate the handling logic for each tree node into a
separate method.

### Preprocessing Lexer

Preprocessing Lexer is the lexer that separates tokens of preprocessor
directives and normal Objective-C code. `DEFAULT_MODE` is used for the
normal code tokens and `DIRECTIVE_MODE` — for the code of directives.
Tokens from DEFAULT_MODE are provided below.

```ANTLR
SHARP:                    '#'                                        -> mode(DIRECTIVE_MODE);
COMMENT:                  '/*' .*? '*/'                              -> type(CODE);
LINE_COMMENT:             '//' ~[\r\n]*                              -> type(CODE);
SLASH:                    '/'                                        -> type(CODE);
CHARACTER_LITERAL:        '\'' (EscapeSequence | ~('\''|'\\')) '\''  -> type(CODE);
QUOTE_STRING:             '\'' (EscapeSequence | ~('\''|'\\'))* '\'' -> type(CODE);
STRING:                   StringFragment                             -> type(CODE);
CODE:                     ~[#'"/]+;
```

If you have a look at this code fragment you may think that some
additional tokens are required (`COMMENT`, `QUOTE_STRING`, and others),
while the Objective-C code uses only one token — `CODE`. The truth is that
the `#` character can be incorporated inside the ordinary strings and
comments. Therefore, these tokens must be defined separately. But this
is not a problem as their type is changed to `CODE` and the preprocessing
parser uses the following rules to separate tokens:

```ANTLR
text
    : code
    | SHARP directive (NEW_LINE | EOF)
    ;

code
    : CODE+
    ;
```

### Preprocessing Parser

Parser separating tokens of the Objective-C code and processing tokens
of preprocessor directives. The result parse tree is then passed to the
preprocessor.

### Preprocessor

Visitor that calculates values of preprocessor directives. Each method
for node traversal returns a string. If the calculated value for the
directive is set to `true`, the subsequent fragment of Objective-C code is
returned. Otherwise, the Objective-C code is replaced with the space
characters. As previously mentioned, it is necessary to maintain the
correct position of the tokens in the core code. To facilitate
understanding, we give the following code fragment in Objective-C:

```Objective-C
BOOL trueFlag =
#if DEBUG
    YES
#else
    arc4random_uniform(100) > 95 ? YES : NO
#endif
;
```

If we use the two-step processing, this fragment will be converted to
the following code in Objective-C with a given conditional symbol `DEBUG`.

```Objective-C
BOOL trueFlag =
         
    YES
     
                                            
      
;
```

Note that all the directives and uncompilable code were replaced with
the space characters. The directives can also be nested:

```Objective-C
#if __IPHONE_OS_VERSION_MIN_REQUIRED >= 60000
    #define MBLabelAlignmentCenter NSTextAlignmentCenter
#else
    #define MBLabelAlignmentCenter UITextAlignmentCenter
#endif
```

### Lexer

Lexer for normal Objective-C code without tokens recognizing
preprocessor directives. If the source file does not contain directives
then the same source file is passed as input.

### Parser

Parser for a normal Objective-C code. The grammar of the parser matches
the one used for one-step processing.

## Other Processing Approaches

There are other ways of parsing preprocessor directives, for example,
you can use a [lexerless parser](https://en.wikipedia.org/wiki/Scannerless_parsing). In theory,
such a parser can combine the advantages of both one-step and two-step
processing; namely, the parser will evaluate the values for the
directives and identify the uncompilable code fragments in a single
iteration. However, these parsers also have disadvantages: they are more
difficult to understand and debug.

We left these solutions without consideration as ANTLR is mostly used
for tokenization. There is already the possibility to create lexerless
grammars and it will be further developed in the future (see the
[discussion](https://github.com/antlr/antlr4/issues/814)).

## Conclusion

In this article, we reviewed the techniques for handling preprocessor
directives, which can be used when parsing C-like languages. These
approaches were already implemented for the Objective-C code and used in
web services such as [**Swiftify**](https://objectivec2swift.com) and
Codebeat. The two-step parser was tested with 20 projects, and more than
95% of all files were processed properly. In addition, one-step processing 
is also implemented to parse the C# code and available in Open Source:
[C# Grammar](https://github.com/antlr/grammars-v4/tree/master/csharp).

Swiftify uses one-step processing for preprocessor directives as our goal is to translate
preprocessor directives to the appropriate language constructs in Swift
(instead of doing the preprocessor’s job), despite the potential parsing
errors. For example, `#define` directives in Objective-C are usually used
to declare global constants and macros. Swift uses constants (**let**) and
functions (**func**) for the same purpose.