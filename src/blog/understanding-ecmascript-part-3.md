---
title: 'Understanding the ECMAScript spec, part 3'
author: '[Marja Hölttä](https://twitter.com/marjakh), speculative specification spectator'
avatars:
  - marja-holtta
date: 2020-03-02 13:33:37
tags:
  - ECMAScript
description: 'Tutorial on reading the ECMAScript specification'
tweet: ''
---

... where we dive deep in the syntax!

## Previous episodes

In [part 2](/blog/understanding-ecmascript-part-2), we examined a simple grammar production and how its runtime semantics are defined. In [the extra content](/blog/extra/understanding-ecmascript-part-2-extra), we also followed a long grammar production chain from `AssignmentExpression` to `MemberExpression`. In this episode, we'll go deeper in the definition of the ECMAScript (or JavaScript) language and its syntax.

If you're not familiar with [context-free grammars](https://en.wikipedia.org/wiki/Context-free_grammar), now it's a good idea to check out the basics, since the spec uses context-free grammars to define the language.

## ECMAScript grammars

ECMAScript source text is a sequence of Unicode code points. Each Unicode code point is an integral value between `U+0000` and `U+10FFFF`. The actual encoding (for example, UTF-8 or UTF-16) is not important &mdash; we assume that the source code has already been converted into a sequence of Unicode code points according to the encoding it was in.

The spec contains several grammars which we'll briefly describe next.

### Lexical grammar

The [lexical grammar](https://tc39.es/ecma262/#sec-ecmascript-language-lexical-grammar) describes how Unicode code points are translated into a sequence of **input elements** (tokens, line terminators, comments, white space).

There are several cases where the next token cannot be identified purely by looking at the Unicode code point stream, but we need to know where we are in the syntactic grammar. A classic example is `/`. To know whether it's a division or the start of the RegExp, we need to know which one is allowed in the syntactic context we're currently in.

For example:
```javascript
const x = 10 / 5;
//           ^ this is a DivPunctuator

const r = /foo/;
//        ^ this is the start of a RegularExpressionLiteral
```

A similar thing happens with templates &mdash; the interpretation of <code>}`</code> depends on the context we're in:

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

The lexical grammar uses several goal symbols to distinguish between the contexts where some input elements are permitted and some are not. For example, the goal symbol `InputElementDiv` is used in contexts where `/` is a division and `/=` is a division-assignment. The `InputElementDiv` productions list the possible tokens which can be produced in this context:

> [`InputElementDiv ::`](https://tc39.es/ecma262/#prod-InputElementDiv)
> `WhiteSpace`
> `LineTerminator`
> `Comment`
> `CommonToken`
> `DivPunctuator`
> `RightBracePunctuator`

In this context, encountering `/` will produce the `DivPunctuator` input element. Producing a `RegularExpressionLiteral` is not an option here.

On the other hand, `InputElementRegExp` is the goal symbol for the contexts where `/` is the beginning of a RegExp:

> [`InputElementRegExp ::`](https://tc39.es/ecma262/#prod-InputElementRegExp)
> `WhiteSpace`
> `LineTerminator`
> `Comment`
> `CommonToken`
> `RightBracePunctuator`
> `RegularExpressionLiteral`

As we see from the productions, it's possible that this produces the `RegularExpressionLiteral` input element, but producing `DivPunctuator` is not possible.

Similarly, there is another goal symbol, `InputElementRegExpOrTemplateTail`, for contexts where `TemplateMiddle` and `TemplateTail` are permitted, in addition to `RegularExpressionLiteral`. And finally, `InputElementTemplateTail` is the goal symbol for contexts where only `TemplateMiddle` and `TemplateTail` are permitted but `RegularExpressionLiteral` is not permitted.

We can imagine the syntactic grammar analyzer ("parser") calling the lexical grammar analyzer ("tokenizer" or "lexer"), passing the goal symbol as a parameter and asking for the next input element suitable for that goal symbol.

### Other grammars

The [RegExp grammar](https://tc39.es/ecma262/#sec-patterns) describes how Unicode code points are translated into regular expressions.

We can imagine the parser asking the tokenizer for the next token in a context where RegExps are allowed. If the tokenizer returns `RegularExpressionLiteral`, we branch into the RegExp grammar for converting the string of the `RegularExpressionLiteral` into a RegExp pattern.

The [numeric string grammar](https://tc39.es/ecma262/#sec-tonumber-applied-to-the-string-type) describes how Strings are translated into numeric values.

The [syntactic grammar](https://tc39.es/ecma262/#sec-syntactic-grammar) describes how syntactically correct programs are composed of tokens.

The notation used for different grammars differs slightly. For example, the syntactic grammar uses `Symbol :` whereas the lexical grammar and the RegExp grammar use `Symbol ::` and the numeric string grammar uses `Symbol :::`.

For the rest of this episode, we'll focus on the syntactic grammar.

## Example: Allowing legacy identifiers

In some contexts, `await` and `yield` are allowed identifiers. Finding out when exactly they are allowed can be a bit involved, so let's dive right in!

Let's have a closer look at allowing `await` as an identifier (`yield` works similarly).

For example, this code works:

```javascript
function my_non_async_function() {
  var await;
  console.log(await);
}
```

However, if we're inside an async function, `await` is treated as a keyword. So this code doesn't work:

```javascript
async function my_async_function() {
  var await; // Syntax error
}
```

### Productions and shorthands

Let's look at how the productions for `VariableStatement` are defined. At the first glance, the grammar can look a bit scary:

> [<code>VariableStatement<sub>[Yield, Await]</sub> :</code>](https://tc39.es/ecma262/#prod-VariableStatement)
> <code>var VariableDeclarationList<sub>[+In, ?Yield, ?Await]</sub>;</code>

What do the subscripts (`[Yield, Await]`) and prefixes (`+` in `+In` and `?` in `?Async`) mean?

The notation is explained in section [Grammar Notation](https://tc39.es/ecma262/#sec-grammar-notation).

The subscripts are a shorthand for expressing a set of productions, for a set of left-hand side symbols, all at once. The left-hand side symbol has two parameters, so the "real" left-hand side symbols we're defining are `VariableStatement`, `VariableStatement_Yield`, `VariableStatement_Await` and `VariableStatement_Yield_Await`.

Note that here the plain `VariableStatement` means "`VariableStatement` without `_Await` and `_Yield`". It should not be confused with <code>VariableStatement<sub>[Yield, Await]</sub></code>.

On the right-hand side of the production, we see the shorthand `+In`, meaning "use the version with `_In`", and `?Await`, meaning "use the version with `_Await` iff the left-hand side symbol has `_Await`" (similarly with `?Yield`).

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

Ultimately, we'll need to find out two things:
1. Where is it decided whether we're in the case with `_Await` or without `_Await`?
1. Where does it make a difference &mdash; where do the productions for `Something_Await` and `Something` (without `_Await`) diverge?

### `_Await` or no `_Await`?

Let's tackle question 1 first. It's somewhat easy to guess that non-async functions and async functions differ in whether we pick the parameter `_Await` for the function body or not. Reading the productions for async function declarations, we find this:

> [`AsyncFunctionBody :`](https://tc39.es/ecma262/#prod-AsyncFunctionBody)
> <code>FunctionBody<sub>[~Yield, +Await]</sub></code>

Note that `AsyncFunctionBody` has no parameters &mdash; they get added to the `FunctionBody` on the right-hand side.

If we expand this production, we get:

> `AsyncFunctionBody :`
> `FunctionBody_Await`

Since `FunctionBody_Await` is used for async functions. It means a function body where `await` is treated as a keyword.

On the other hand, if we're inside a non-async function, the relevant production is:

> [<code>FunctionDeclaration<sub>[Yield, Await, Default]</sub>](https://tc39.es/ecma262/#prod-FunctionDeclaration) :</code>
> <code>function BindingIdentifier<sub>[?Yield, ?Await]</sub> ( FormalParameters<sub>[~Yield, ~Await]</sub> ) { FunctionBody<sub>[~Yield, ~Await]</sub> }</code>

(`FunctionDeclaration` has another production, but it's not relevant for our code example.)

To avoid combinatorial expansion, let's ignore the `Default` parameter which is not used in this particular production.

The expanded form of the production is:

> `FunctionDeclaration :`
> `function BindingIdentifier ( FormalParameters ) { FunctionBody }`

> `FunctionDeclaration_Yield :`
> `function BindingIdentifier_Yield ( FormalParameters_Yield ) { FunctionBody }`

> `FunctionDeclaration_Await :`
> `function BindingIdentifier_Await ( FormalParameters_Await ) { FunctionBody }`

> `FunctionDeclaration_Yield_Await :`
> `function BindingIdentifier_Yield_Await ( FormalParameters_Yield_Await ) { FunctionBody }`

In this production we always get `FunctionBody` (without `_Yield` and without `_Await`), since the `FunctionBody` in the non-expanded production is parameterized with `[~Yield, ~Await]`.

Function name and formal parameters are treated differently: they get the parameters `_Await` and `_Yield` if the left-hand side symbol has them.

To summarize: Async functions have a `FunctionBody_Await` and non-async functions have a `FunctionBody` (without `_Await`). You can think of the `_Await` parameter meaning "`await` is a keyword".

Since we're talking about non-generator functions, both our async example function and our non-async example function are parameterized without `_Yield`.

### Disallowing `await` as an identifier

Next, we need to find out how `await` is disallowed as an identifier if we're inside a `FunctionBody_Await`.

We can follow the productions further to see that the `_Await` parameter gets carried unchanged from `FunctionBody` all the way to the `VariableStatement` production we were previously looking at.

Thus, inside an async function, we'll have a `VariableStatement_Await` and inside a non-async function, we'll have a `VariableStatement`.

We can follow the productions further and keep track of the parameters. We already saw the productions for `VariableStatement`:

> [<code>VariableStatement<sub>[Yield, Await]</sub> :</code>](https://tc39.es/ecma262/#prod-VariableStatement)
> <code>var VariableDeclarationList<sub>[+In, ?Yield, ?Await]</sub>;</code>

All productions for `VariableDeclarationList` just carry the parameters on as is:

> [<code>VariableDeclarationList<sub>[In, Yield, Await]</sub> :</code>](https://tc39.es/ecma262/#prod-VariableDeclarationList)
> <code>VariableDeclaration<sub>[?In, ?Yield, ?Await]</sub></code>

(Here we show only the production relevant to our example.)

> [<code>VariableDeclaration<sub>[In, Yield, Await]</sub> :</code>](https://tc39.es/ecma262/#prod-VariableDeclaration)
> <code>BindingIdentifier<sub>[?Yield, ?Await]</sub> Initializer<sub>[?In, ?Yield, ?Await]</sub> opt</code>

The `opt` shorthand means that the right-hand side symbol is optional; there are in fact two productions, one with the optional symbol, and one without.

In the simple case relevant to our example, `VariableStatement` consists of the keyword `var`, followed by a single `BindingIdentifier` without an initializer, and ending with a semicolon.

To disallow or allow `await` as a `BindingIdentifier`, we hope to end up with something like this:

> `BindingIdentifier_Await :`
> `Identifier`
> `yield`
>
> `BindingIdentifier :`
> `Identifier`
> `yield`
> `await`

This would disallow `await` as an identifier inside async functions and allow it as an identifier inside non-async functions.

But the spec doesn't define it like this, instead we find this production:

> [<code>BindingIdentifier<sub>[Yield, Await]</sub> :</code>](https://tc39.es/ecma262/#prod-BindingIdentifier)
> `Identifier`
> `yield`
> `await`

Expanded, this means the following productions:

> `BindingIdentifier_Await :`
> `Identifier`
> `yield`
> `await`
>
> `BindingIdentifier :`
> `Identifier`
> `yield`
> `await`

(We're omitting the productions for `BindingIdentifier_Yield` and `BindingIdentifier_Yield_Await` which are not needed in our example.)

This looks like `await` and `yield` would be always allowed as identifiers. What's up with that? Is the whole blog post for nothing?

### Statics semantics to the rescue

Turns out **static semantics** are needed for forbidding `await` as an identifier inside async functions.

Static semantics describe static rules &mdash; that is, rules that can be checked before the program is ran.

In this case, the [static semantics for BindingIdentifier](https://tc39.es/ecma262/#sec-identifiers-static-semantics-early-errors) define the following syntax-directed rule:

>`BindingIdentifier : await`
>
> It's a Syntax Error if this production has an <code><sub>[Await]</sub></code> parameter.

Effectively, this forbids the `BindingIdentifier_Await : await` production.

The spec explains that the reason for having this production but defining it as a Syntax Error by the static semantics is because of interference with automatic semicolon insertion. If the production was missing, automatic semicolon insertion might kick in and insert a semicolon into a program which is syntactically incorrect only because it uses `await` or `yield` as an identifier, changing the meaning of the program.

There's also another related rule:

> `BindingIdentifier : Identifier`
>
> It is a Syntax Error if this production has an <code><sub>[Await]</sub></code> parameter and `StringValue` of `Identifier` is `"await"`.

This might be confusing at first. `Identifier` is defined like this:

> [`Identifier :`](https://tc39.es/ecma262/#prod-Identifier)
> `IdentifierName` but not `ReservedWord`

`await` is a `ReservedWord`, so how can an `Identifier` ever be `await`?

Turns out, `Identifier` cannot be `await`, but it can be something else whose `StringValue` is `"await"` &mdash; a different representation of the character sequence `await`. [Static semantics for identifier names](https://tc39.es/ecma262/#sec-identifier-names-static-semantics-stringvalue) define how the `StringValue` of an identifier name is computed.

For example, the Unicode escape sequence for `a` is `\0061`, so `\u0061wait` has the `StringValue` `"await"`. `\u0061wait` won't be recognized as a keyword by the lexical grammar, instead it will be an `Identifier`. The static semantics for forbid using it as a variable name inside async functions.

So this works:

```javascript
function my_non_async_function() {
  var \0061wait;
  console.log(await);
}
```

And this doesn't:

```javascript
async function my_async_function() {
  var \0061wait; // Syntax error
}
```

## Summary

In this episode, we familiarized ourselves with the ECMAScript syntactic grammar and the shorthands used for defining the productions. As an example, we looked into forbidding using `await` as an identifier inside async functions but allowing it inside non-async functions.
