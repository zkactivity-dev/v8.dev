---
title: 'Understanding the ECMAScript spec, part 3'
author: '[Marja Hölttä](https://twitter.com/marjakh), speculative specification spectator'
avatars:
  - marja-holtta
date: 2020-02-11 13:33:37
tags:
  - ECMAScript
description: 'Tutorial on reading the ECMAScript specification'
tweet: ''
---

... where we dive deep in the syntax!

## Previous episodes

In part 2, we briefly looked at a simple grammar production and how its runtime semantics are defined.  We also followed a long grammar production chain from `AssignmentExpression` to `MemberExpression`. In this episode, we go deeper in the definition of the ECMAScript (or JavaScript) language and its syntax.

If you're not familiar with [context-free grammars](https://en.wikipedia.org/wiki/Context-free_grammar), now it's a good idea to check out the basics, since the spec uses context-free grammars to define the language.

## ECMAScript grammars

ECMAScript source text is a sequence of Unicode code points. Each Unicode code point is an integral value between U+0000 and U+10FFFF. The actual encoding (for example, UTF-8, UTF-16 and so on) is not relevant for the spec; we assume that the source code has already been converted into a sequence of Unicode code points according to the encoding it was in.

The spec contains several grammars which we'll briefly describe next.

The [lexical grammar](https://tc39.es/ecma262/#sec-ecmascript-language-lexical-grammar) describes how Unicode code points are translated into a sequence of **input elements** (tokens, line terminators, comments, white space).

There are several cases where the next token cannot be identified purely by looking at the Unicode code point stream, but we need to know where we are in the syntactic grammar. A classic example is `/`. To know whether it's the division or the start of the RegExp, we need to know which one is allowed in the syntactic context we're currently in.

For example:
```javascript
const x = 10 / 5; // Here / is DivPunctuator

const r = /foo/; // Here / is the start of a RegularExpressionLiteral
```

A similar thing happens with templates &mdash; the interpretation of <code>{`</code> depends on the context we're in:

```javascript
const what1 = 'temp';
const what2 = 'late';
const t = `I am a ${ what1 + what2 }`;
// `I am a ${ is TemplateHead
// }` is TemplateTail

if (0 == 1) {
}`not very useful`;
// } is RightBracePunctuator
// ` is the start of a NoSubstitutionTemplate

```

The lexical grammar uses several goal symbols to distinguish between the contexts where some input elements are permitted and some are not. For example, the goal symbol `InputElementDiv` is used in contexts where `/` is a division and `/=` is a division-assignment:

> [`InputElementDiv ::`](https://tc39.es/ecma262/#prod-InputElementDiv)
> `WhiteSpace`
> `LineTerminator`
> `Comment`
> `CommonToken`
> `DivPunctuator`
> `RightBracePunktuator`

In this context, encountering `/` will produce the `DivPunctuator` input element. Producing a `RegularExpressionLiteral` is not an option here.

On the other hand, `InputElementRegExp` is a goal symbol for the contexts where `/` is the beginning of a RegExp:

> [`InputElementRegExp ::`](https://tc39.es/ecma262/#prod-InputElementRegExp)
> `WhiteSpace`
> `LineTerminator`
> `Comment`
> `CommonToken`
> `RightBracePunktuator`
> `RegularExpressionLiteral`

As we see from the productions, it's possible that this produces the `RegularExpressionLiteral` input element, but producing `DivPunctuator` is not possible.

Similarly, there is another goal symbol, `InputElementRegExpOrTemplateTail`, for contexts where `TemplateMiddle` and `TemplateTail` are permitted, in addition to `RegularExpressionLiteral`. And finally, `InputElementTemplateTail` is the goal symbol for contexts where only `TemplateMiddle` and `TemplateTail` are permitted but `RegularExpressionLiteral` is not permitted.

We can imagine the syntactic grammar analyzer ("parser") calling the lexical grammar analyzer ("tokenizer" or "lexer"), passing the goal symbol as a parameter and asking for the next input element suitable for that goal symbol.

The [RegExp grammar](https://tc39.es/ecma262/#sec-patterns) describes how Unicode code points are translated into regular expressions.

We can imagine the parser asking the tokenizer for the next token in a context where RegExps are allowed. If the tokenizer returns `RegularExpressionLiteral`, we branch into the RegExp grammar for converting the string of the `RegularExpressionLiteral` into a RegExp pattern.

The [numeric string grammar](https://tc39.es/ecma262/#sec-tonumber-applied-to-the-string-type) describes how Strings are translated into numeric values.

The [syntactic grammar](https://tc39.es/ecma262/#sec-syntactic-grammar) describes how syntactically correct programs are composed of tokens.

The notation used for different grammars differs slightly. For example, the syntactic grammar uses `Symbol :` whereas the lexical grammar use and the RegExp grammar use `Symbol ::` and the numeric string grammar uses `Symbol :::`.

For the rest of this episode, we'll focus on the syntactic grammar.

## Example: Allowing legacy identifiers

In some contexts, `await` and `yield` are allowed identifiers. Finding out when exactly they are allowed can be a bit involved, so let's dive right in!

Let's have a closer look at allowing `await` as an identifier (`yield` works similarly).

For example, this code works:

```javascript
function old() {
  var await = 900;
  console.log(await);
}
```

but this doesn't:

```javascript
async function modern() {
  var await = 900; // Syntax error
}
```

However, if we're inside an async function, `await` is treated as a keyword. This is not a breaking change: if a developer writes an async function, they are already using a newer version of the language and there's no reason to allow `await` as an identifier.

At the first glance, the grammar rules can look a bit scary:

> <code>VariableStatement<sub>[Yield, Await]</sub> :</code>
> <code>var VariableDeclarationList<sub>[+In, ?Yield, ?Await]</sub>;</code>

Huh? What are the weird subscripts? We have not only `[Yield, Await]` but also `+` in `+In` and the `?` in `?Await`. What does all that mean?

The notation is explained in section [Grammar Notation](https://tc39.es/ecma262/#sec-grammar-notation).

The subscripts are a shorthand for expressing a set of productions, for a set of left-hand side symbols, all at once. The left-hand side symbol has two parameters, so the "real" left-hand side symbols we're defining are `VariableStatement`, `VariableStatement_Yield`, `VariableStatement_Await` and `VariableStatement_Yield_Await`.

On the right hand side of the production, we see the shorthand `+In`, meaning "use the version with `_In`", and `+Await`, meaning "use the version with `_Await` iff the left-hand side symbol has `_Await` (similarly with `?Yield`).

(The third shorthand, `~Foo`, meaning "use the version without `_Foo`", is not used in this production.)

With this information, we can expand the productions like this:

> `VariableStatement` :
> `var VariableDeclarationList_In;`
>
> `VariableStatement_Yield` :
> `var VariableDeclarationList_In_Yield;`
>
> `VariableStatement_Await` :
> `var VariableDeclarationList_In_Await;`
>
> `VariableStatement_Yield_Await` :
> `var VariableDeclarationList_In_Yield_Await;`

We can follow the productions further and keep track of the parameters. For example, all productions for `VariableDeclarationList` just carry them on as is, which is not very interesting:

> <code>VariableDeclarationList<sub>[In, Yield, Await]</sub> :</code>
> <code>VariableDeclaration<sub>[?In, ?Yield, ?Await]</sub></code>
> <code>VariableDeclarationList<sub>[?In, ?Yield, ?Await]</sub> , VariableDeclaration<sub>[?In, ?Yield, ?Await]</sub></code>

Ultimately, we'll need to know two things: 1) Where is it decided whether we're in the case with `_Await` or without `_Await`? 2) Where does it make a difference &mdash; where do the productions for `Something_Await` and `Something` (without `_Await`) diverge?

Let's tackle question 1 first. It's somewhat easy to guess it makes a difference in async functions. Reading the rules for async function declarations, we find this:

> `AsyncFunctionBody :`
> <code>FunctionBody<sub>[~Yield, +Await]</sub></code>

Note that `AsyncFunctionBody` has no parameters, but they're introduced here.

If we expand this production, we get:

> `AsyncFunctionBody :`
> `FunctionBody_Await`

On the other hand, if we're inside a normal, non-async function, the relevant production is:

> <code>FunctionDeclaration<sub>[Yield, Await, Default]</sub> :</code>
> <code>function BindingIdentifier<sub>[?Yield, ?Await]</sub> ( FormalParameters<sub>[~Yield, ~Await]</sub> ) { FunctionBody<sub>[~Yield, ~Await]</sub> }</code>

(`FunctionDeclaration` has another production, but it's not relevant for our code example.)

To avoid combinatorial expansion, let's ignore the `Default` parameter which is not relevant for this particular production.

The expanded form of the production is:

> `FunctionDeclaration :`
> `function BindingIdentifier ( FormalParameters ) { FunctionBody }`

> `FunctionDeclaration_Yield :`
> `function BindingIdentifier_Yield ( FormalParameters_Yield ) { FunctionBody }`

> `FunctionDeclaration_Await :`
> `function BindingIdentifier_Await ( FormalParameters_Await ) { FunctionBody }`

> `FunctionDeclaration_Yield_Await :`
> `function BindingIdentifier_Yield_Await ( FormalParameters_Yield_Await ) { FunctionBody }`

The important thing in this production is that `FunctionBody` is parameterized with `[~Yield, ~Await]`, meaning that we always take the version without `_Yield` and without `_Await`, no matter what the parameters of `FunctionDeclaration` were.

We can follow the productions further to see that they get carried unchanged from `AsyncFunctionBody` and `FunctionBody` all the way to the `VariableStatement` production we were previously looking at.
