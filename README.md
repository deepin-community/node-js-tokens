# js-tokens

The tiny, regex powered, lenient, _almost_ spec-compliant JavaScript tokenizer that never fails.

```js
const jsTokens = require("js-tokens");

const jsString = 'JSON.stringify({k:3.14**2}, null /*replacer*/, "\\t")';

Array.from(jsTokens(jsString), (token) => token.value).join("|");
// JSON|.|stringify|(|{|k|:|3.14|**|2|}|,| |null| |/*replacer*/|,| |"\t"|)
```

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Installation](#installation)
- [Usage](#usage)
- [Tokens](#tokens)
  - [StringLiteral](#stringliteral)
  - [NoSubstitutionTemplate / TemplateHead / TemplateMiddle / TemplateTail](#nosubstitutiontemplate--templatehead--templatemiddle--templatetail)
  - [RegularExpressionLiteral](#regularexpressionliteral)
  - [MultiLineComment](#multilinecomment)
  - [SingleLineComment](#singlelinecomment)
  - [IdentifierName](#identifiername)
  - [NumericLiteral](#numericliteral)
  - [Punctuator](#punctuator)
  - [WhiteSpace](#whitespace)
  - [LineTerminatorSequence](#lineterminatorsequence)
  - [Invalid](#invalid)
- [JSX Tokens](#jsx-tokens)
  - [JSXString](#jsxstring)
  - [JSXText](#jsxtext)
  - [JSXIdentifier](#jsxidentifier)
  - [JSXPunctuator](#jsxpunctuator)
  - [JSXInvalid](#jsxinvalid)
- [Compatibility](#compatibility)
  - [ECMAScript](#ecmascript)
    - [Annex B](#annex-b)
  - [TypeScript](#typescript)
  - [JSX](#jsx)
  - [JavaScript runtimes](#javascript-runtimes)
  - [Known errors](#known-errors)
- [Performance](#performance)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Installation

`npm install js-tokens`

```js
import jsTokens from "js-tokens";
// or:
const jsTokens = require("js-tokens");
```

## Usage

```js
jsTokens(string, options?)
```

| Option | Type      | Default | Description         |
| :----- | :-------- | :------ | :------------------ |
| jsx    | `boolean` | `false` | Enable JSX support. |

This package exports a generator function, `jsTokens`, that turns a string of JavaScript code into token objects.

For the empty string, the function yields nothing (which can be turned into an empty list). For any other input, the function always yields _something,_ even for invalid JavaScript, and never throws. Concatenating the token values reproduces the input.

The package is very close to being fully spec compliant (it passes all [but 3](#known-errors) of [test262-parser-tests]), but has taken a couple of shortcuts. See the following sections for limitations of some tokens.

```js
// Loop over tokens:
for (const token of jsTokens("hello, !world")) {
  console.log(token);
}

// Get all tokens as an array:
const tokens = Array.from(jsTokens("hello, !world"));
```

## Tokens

_Spec: [ECMAScript Language: Lexical Grammar] + [Additional Syntax]_

```ts
export default function jsTokens(input: string): Iterable<Token>;

type Token =
  | { type: "StringLiteral"; value: string; closed: boolean }
  | { type: "NoSubstitutionTemplate"; value: string; closed: boolean }
  | { type: "TemplateHead"; value: string }
  | { type: "TemplateMiddle"; value: string }
  | { type: "TemplateTail"; value: string; closed: boolean }
  | { type: "RegularExpressionLiteral"; value: string; closed: boolean }
  | { type: "MultiLineComment"; value: string; closed: boolean }
  | { type: "SingleLineComment"; value: string }
  | { type: "IdentifierName"; value: string }
  | { type: "NumericLiteral"; value: string }
  | { type: "Punctuator"; value: string }
  | { type: "WhiteSpace"; value: string }
  | { type: "LineTerminatorSequence"; value: string }
  | { type: "Invalid"; value: string };
```

### StringLiteral

_Spec: [StringLiteral]_

If the ending `"` or `'` is missing, the token has `closed: false`. JavaScript strings cannot contain (unescaped) newlines, so unclosed strings simply end at the end of the line.

Escape sequences are supported, but may be invalid. For example, `"\u"` is matched as a StringLiteral even though it contains an invalid escape.

Examples:

<!-- prettier-ignore -->
```js
"string"
'string'
""
''
"\""
'\''
"valid: \u00a0, invalid: \u"
'valid: \u00a0, invalid: \u'
"multi-\
line"
'multi-\
line'
" unclosed
' unclosed
```

### NoSubstitutionTemplate / TemplateHead / TemplateMiddle / TemplateTail

_Spec: [NoSubstitutionTemplate] / [TemplateHead] / [TemplateMiddle] / [TemplateTail]_

A template without interpolations is matched as is. For, example:

- `` `abc` ``: NoSubstitutionTemplate
- `` `abc ``: NoSubstitutionTemplate with `closed: false`

A template _with_ interpolations is matched as many tokens. For example, `` `head${1}middle${2}tail` `` is matched as follows (apart from the two NumericLiterals):

- `` `head${ ``: TemplateHead
- `}middle${`: TemplateMiddle
- `` }tail` ``: TemplateTail

TemplateMiddle is optional, and TemplateTail can be unclosed. For example, `` `head${1}tail `` (note the missing ending `` ` ``):

- `` `head${ ``: TemplateHead
- `}tail`: TemplateTail with `closed: false`

Templates can contain unescaped newlines, so unclosed templates go on to the end of input.

Just like for StringLiteral, templates can also contain invalid escapes. `` `\u` `` is matched as a NoSubstitutionTemplate even though it contains an invalid escape. Also note that in _tagged_ templates, invalid escapes are _not_ syntax errors: `` x`\u` `` is syntactically valid JavaScript.

### RegularExpressionLiteral

_Spec: [RegularExpressionLiteral]_

Regex literals may contain invalid regex syntax. They are still matched as regex literals.

If the ending `/` is missing, the token has `closed: false`. JavaScript regex literals cannot contain newlines (not even escaped ones), so unclosed regex literals simply end at the end of the line.

According to the specification, the flags of regular expressions are [IdentifierPart]s (unknown and repeated regex flags become errors at a later stage).

Differentiating between regex and division in JavaScript is really tricky. js-tokens looks at the previous token to tell them apart. As long as the previous tokens are valid, it should do the right thing. For invalid code, js-tokens might be confused and start matching division as regex or vice versa.

Examples:

<!-- prettier-ignore -->
```js
/a/
/a/gimsuy
/a/Inva1id
/+/
/[/]\//
```

### MultiLineComment

_Spec: [MultiLineComment]_

If the ending `*/` is missing, the token has `closed: false`. Unclosed multi-line comments go on to the end of the input.

Examples:

<!-- prettier-ignore -->
```js
/* comment */
/* console.log(
    "commented", out + code);
    */
/**/
/* unclosed
```

### SingleLineComment

_Spec: [SingleLineComment]_

Examples:

<!-- prettier-ignore -->
```js
// comment
// console.log("commented", out + code);
//
```

### IdentifierName

_Spec: [IdentifierName]_

Keywords, reserved words, `null`, `true`, `false`, variable names and property names.

Examples:

<!-- prettier-ignore -->
```js
if
for
var
instanceof
package
null
true
false
Infinity
undefined
NaN
$variab1e_name
??
???
???_???
\u006C\u006F\u006C\u0077\u0061\u0074
```

### NumericLiteral

_Spec: [NumericLiteral]_

Examples:

<!-- prettier-ignore -->
```js
0
1.5
1
12e9
0.123e-32
0xDeadbeef
0b110
12n
07
09.5
```

### Punctuator

_Spec: [Punctuator] + [DivPunctuator] + [RightBracePunctuator]_

All possible values:

<!-- prettier-ignore -->
```js
&&  ||  ??
--  ++
.   ?.
<   <=   >   >=
!=  !==  ==  ===
   +   -   %   &   |   ^   /   *   **   <<   >>   >>>
=  +=  -=  %=  &=  |=  ^=  /=  *=  **=  <<=  >>=  >>>=
(  )  [  ]  {  }
!  ?  :  ;  ,  ~  ...  =>
```

### WhiteSpace

_Spec: [WhiteSpace]_

Unlike the specification, multiple whitespace characters in a row are matched as _one_ token, not one token per character.

### LineTerminatorSequence

_Spec: [LineTerminatorSequence]_

CR, LF and CRLF, plus `\u2028` and `\u2029`.

### Invalid

_Spec: n/a_

Single code points not matched in another tokens.

Examples:

<!-- prettier-ignore -->
```js
#
@
????
```

## JSX Tokens

_Spec: [JSX Specification]_

```ts
export default function jsTokens(
  input: string,
  options: { jsx: true }
): Iterable<Token | JSXToken>;

export declare type JSXToken =
  | { type: "JSXString"; value: string; closed: boolean }
  | { type: "JSXText"; value: string }
  | { type: "JSXIdentifier"; value: string }
  | { type: "JSXPunctuator"; value: string }
  | { type: "JSXInvalid"; value: string };
```

- The tokenizer switches between outputting runs of `Token` and runs of `JSXToken`.
- Runs of `JSXToken` can also contain WhiteSpace, LineTerminatorSequence, MultiLineComment and SingleLineComment.

### JSXString

_Spec: `"` JSXDoubleStringCharacters `"` + `'` JSXSingleStringCharacters `'`_

If the ending `"` or `'` is missing, the token has `closed: false`. JSX strings can contain unescaped newlines, so unclosed JSX strings go on to the end of input.

Note that JSX don???t support escape sequences as part of the token grammar. A `"` or `'` always closes the string, even with a backslash before.

Examples:

<!-- prettier-ignore -->
```
"string"
'string'
""
''
"\"
'\'
"multi-
line"
'multi-
line'
" unclosed
' unclosed
```

### JSXText

_Spec: JSXText_

Anything but `<`, `>`, `{` and `}`.

### JSXIdentifier

_Spec: JSXIdentifier_

Examples:

<!-- prettier-ignore -->
```js
div
class
xml
x-element
x------
$htm1_element
???_???
```

### JSXPunctuator

_Spec: n/a_

All possible values:

```js
<
>
/
.
:
=
{
}
```

### JSXInvalid

_Spec: n/a_

Single code points not matched in another token.

Examples in JSX tags:

```js
1
`
+
,
#
@
????
```

All possible values in JSX children:

```js
>
}
```

## Compatibility

### ECMAScript

The intention is to always support the latest ECMAScript version whose feature set has been finalized.

Currently, ECMAScript 2020 is supported.

#### Annex B

[Annex B: Additional ECMAScript Features for Web Browsers][annexb] of the spec is optional if the ECMAScript host is not a web browser, and specifies some additional syntax.

- Numeric literals: js-tokens supports legacy octal and octal like numeric literals. It was easy enough, so why not.
- String literals: js-tokens supports legacy octal escapes, since it allows any invalid escapes.
- HTML-like comments: **Not supported.** js-tokens prefers treating `5<!--x` as `5 < !(--x)` rather than as `5 //x`.
- Regular expression patterns: js-tokens doesn???t care what???s between the starting `/` and ending `/`, so this is supported.

### TypeScript

Supporting TypeScript is not an explicit goal, but js-tokens and Babel both tokenize this [TypeScript fixture] and this [TSX fixture] the same way, with one edge case:

<!-- prettier-ignore -->
```ts
type A = Array<Array<string>>
type B = Array<Array<Array<string>>>
```

Both lines above should end with a couple of `>` tokens, but js-tokens instead matches the `>>` and `>>>` operators.

### JSX

JSX is supported: `jsTokens("<p>Hello, world!</p>", { jsx: true })`.

### JavaScript runtimes

js-tokens should work in any JavaScript runtime that supports [Unicode property escapes]. For Node.js, this means Node.js 10 or later.

### Known errors

Here are a couple of tricky cases:

<!-- prettier-ignore -->
```js
// Case 1:
switch (x) {
  case x: {}/a/g;
  case x: {}<div>x</div>/g;
}

// Case 2:
label: {}/a/g;
label: {}<div>x</div>/g;

// Case 3:
(function f() {}/a/g);
(function f() {}<div>x</div>/g);
```

This is what they mean:

```js
// Case 1:
switch (x) {
  case x:
    {
    }
    /a/g;
  case x:
    {
    }
    <div>x</div> / g;
}

// Case 2:
label: {
}
/a/g;
label: {
}
<div>x</div> / g;

// Case 3:
(function f() {} / a / g);
(function f() {} < div > x < /div>/g);
```

But js-tokens thinks they mean:

```js
// Case 1:
switch (x) {
  case x:
    ({} / a / g);
  case x:
    ({} < div > x < /div>/g);
}

// Case 2:
label: ({} / a / g);
label: ({} < div > x < /div>/g);

// Case 3:
function f() {}
/a/g;
function f() {}
<div>x</div> / g;
```

In other words, js-tokens:

- Mis-identifies regex as division and JSX as comparison in case 1 and 2.
- Mis-identifies division as regex and comparison as JSX in case 3.

This happens because js-tokens looks at the previous token when deciding between regex and division or JSX and comparison. In these cases, the previous token is `}`, which either means ???end of block??? (??? regex/JSX) or ???end of object literal??? (??? division/comparison). How does js-tokens determine if the `}` belongs to a block or an object literal? By looking at the token before the matching `{`.

In case 1 and 2, that???s a `:`. A `:` _usually_ means that we have an object literal or ternary:

```js
let some = weird ? { value: {}/a/g } : {}/a/g;
```

But `:` is also used for `case` and labeled statements.

One idea is to look for `case` before the `:` as an exception to the rule, but it???s not so easy:

<!-- prettier-ignore -->
```js
switch (x) {
  case weird ? true : {}/a/g: {}/a/g
}
```

The first `{}/a/g` is a division, while the second `{}/a/g` is an empty block followed by a regex. Both are preceded by a colon with a `case` on the same line, and it does not seem like you can distinguish between the two without implementing a parser.

Labeled statements are similarly difficult, since they are so similar to object literals:

<!-- prettier-ignore -->
```js
{
  label: {}/a/g
}

({
  key: {}/a/g
})
```

Finally, case 3 (`(function f() {}/a/g);`) is also difficult, because a `)` before a `{` means that the `{` is part of a _block,_ and blocks are _usually_ statements:

```js
if (x) {
}
/a/g;

function f() {}
/a/g;
```

But _function expressions_ are of course not statements. It???s difficult to tell an function _expression_ from a function _statement_ without parsing.

Luckily, none of these edge cases are likely to occur in real code.

## Performance

With [@babel/parser] for comparison.

| Lines of code |    Size | js-tokens@6.0.0 | @babel/parser@7.9.4 |
| ------------: | ------: | --------------: | ------------------: |
|          ~100 | ~4.8 KB |           ~2 ms |              ~17 ms |
|        ~1 000 |  ~46 KB |          ~11 ms |              ~84 ms |
|       ~10 000 | ~409 KB |          ~80 ms |             ~550 ms |
|      ~100 000 | ~3.3 MB |         ~430 ms |             ~7.45 s |
|    ~1 500 000 |  ~77 MB |            ~7 s |     ~4 minutes (\*) |

(\*) Required increasing Node.js??? memory limit.

See [benchmark.js] if you want to run benchmarks yourself.

## License

[MIT](LICENSE).

[@babel/parser]: https://babeljs.io/docs/en/babel-parser
[additional syntax]: https://tc39.es/ecma262/#sec-additional-syntax
[annexb]: https://tc39.es/ecma262/#sec-additional-ecmascript-features-for-web-browsers
[benchmark.js]: https://github.com/lydell/js-tokens/blob/master/benchmark.js
[divpunctuator]: https://tc39.es/ecma262/#prod-DivPunctuator
[ecmascript language: lexical grammar]: https://tc39.es/ecma262/#sec-ecmascript-language-lexical-grammar
[example.test.js]: https://github.com/lydell/js-tokens/blob/master/test/example.test.js
[identifiername]: https://tc39.es/ecma262/#prod-IdentifierName
[identifierpart]: https://tc39.es/ecma262/#prod-IdentifierPart
[jsx specification]: https://facebook.github.io/jsx/
[lineterminatorsequence]: https://tc39.es/ecma262/#prod-LineTerminatorSequence
[multilinecomment]: https://tc39.es/ecma262/#prod-MultiLineComment
[nosubstitutiontemplate]: https://tc39.es/ecma262/#prod-NoSubstitutionTemplate
[numericliteral]: https://tc39.es/ecma262/#prod-annexB-NumericLiteral
[punctuator]: https://tc39.es/ecma262/#prod-Punctuator
[regularexpressionliteral]: https://tc39.es/ecma262/#prod-RegularExpressionLiteral
[rightbracepunctuator]: https://tc39.es/ecma262/#prod-RightBracePunctuator
[singlelinecomment]: https://tc39.es/ecma262/#prod-SingleLineComment
[stackoverflow-slash]: https://stackoverflow.com/a/27120110/2010616
[string.prototype.matchall]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/matchAll
[stringliteral]: https://tc39.es/ecma262/#prod-StringLiteral
[templatehead]: https://tc39.es/ecma262/#prod-TemplateHead
[templatemiddle]: https://tc39.es/ecma262/#prod-TemplateMiddle
[templatetail]: https://tc39.es/ecma262/#prod-TemplateTail
[test262-parser-tests]: https://github.com/tc39/test262-parser-tests
[tsx fixture]: https://github.com/lydell/js-tokens/blob/master/test/fixtures/valid/jsx2.tsx
[typescript fixture]: https://github.com/lydell/js-tokens/blob/master/test/fixtures/valid/typescript.module.ts
[unicode property escapes]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions/Unicode_Property_Escapes
[whitespace]: https://tc39.es/ecma262/#prod-WhiteSpace
