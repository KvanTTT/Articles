# Abstract

ANTLR is one of the most popular parser generators and supports a number of runtimes:
Java, C#, Python2, Python3, JavaScript, C++, Swift, and Go. Although ANTLR has
support for context-free grammars, in cases when a language has context
dependence (such as if it's necessary to know the value of a preceding token,
as in [PHP Heredoc](http://php.net/manual/en/language.types.string.php)) or if
certain constructs require more extensive processing (for example, [interpolated strings](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated)),
it is possible to use **code inserts** orsemantic predicates.

Unfortunately, such inserts are not universal because they are written in the
runtime language. In addition, they are not validated at the parser generation
stage. If they were universal and could be translated into the runtime language
at the generation stage, the scope of grammar applicability would expand greatly.
And this in turn would attract more users, inspiring greater development and testing.

Besides **universal code inserts**, it is also possible to broaden the grammar
description language itself, such as with a **token evaluation operator**. This is
useful for languages with a large number of words that are keywords in particular
contexts. The ANTLR lexer description language does not have sufficient syntax
for leaving comments in a convenient way.

A **full-fidelity parse tree** is a concept for associating hidden tokens with nodes
of the parse tree. This is convenient not only for refactoring, but when defects
are found in source code (such as [goto fail](https://nakedsecurity.sophos.com/2014/02/24/anatomy-of-a-goto-fail-apples-ssl-bug-explained-plus-an-unofficial-patch))
that can be detected both by control flow analysis or by incorrect indentation.
The API for the generated code can be improved to make access just as convenient
as in mature specialized code analyzers such as [Roslyn](https://en.wikipedia.org/wiki/.NET_Compiler_Platform).

In addition, ANTLR-generated code often is not idiomatic due to runtime-specific
naming conventions and lacks support for implementing **scannerless** parsers,
which is important for languages in which parsing "errors" do not exist, such as
Markdown.

The article gives a detailed description of these and other problems, as well as
methods for addressing them. The most promising ways forward for ANTLR-based
parser infrastructure, including online viewer and grammar editor, are covered
as well.

A parser generator that has solved these problems and added the described features
may still not be perfect—but it would definitely be pretty close!
