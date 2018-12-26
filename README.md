# engine262

An implementation of ECMA-262 in JavaScript

Goals
- 100% Spec Compliance
- Introspection
- Ease of modification

Non-Goals
- Speed at the expense of any of the goals

This project is bound by a [Code of Conduct][COC].

Join us on [#engine262 on freenode][irc] ([web][irc-webchat]).

## Why this exists

While helping develop new features for JavaScript, I've found that one of the
most useful methods of finding what works and what doesn't is being able to
actually run code using the new feature. [Babel][] is fantastic for this, but
sometimes features just can't be nicely represented with it. Similarly,
implementing a feature in one of the engines is a large undertaking, involving
long compile times and annoying bugs with the optimizing compilers.

engine262 is a tool to allow JavaScript developers to have a sandbox where new
features can be quickly prototyped and explored. As an example, adding
[do expressions][] to this engine is as simple as the following diff:

```diff
--- a/src/evaluator.mjs
+++ b/src/evaluator.mjs
@@ -416,6 +416,9 @@ function* Inner_Evaluate_Expression(Expression) {
     case isExpressionWithComma(Expression):
       return yield* Evaluate_ExpressionWithComma(Expression);

+    case Expression.type === 'DoExpression':
+      return yield* Evaluate_BlockStatement(Expression.body);
+
     default:
       throw new OutOfRange('Evaluate_Expression', Expression);
   }
--- a/src/parse.mjs
+++ b/src/parse.mjs
@@ -11,6 +11,17 @@ const Parser = acorn.Parser.extend((P) => class Parse262 extends P {
     node.source = () => this.input.slice(node.start, node.end);
     return ret;
   }
+
+  parseExprAtom(refDestructuringErrors) {
+    if (this.value === 'do') {
+      // DoExpression : `do` Block
+      this.next();
+      const node = this.startNode();
+      node.body = this.parseBlock();
+      return this.finishNode(node, 'DoExpression');
+    }
+    return super.parseExprAtom(refDestructuringErrors);
+  }
 });
```

This simplicity applies to many other proposals, such as [optional chaining][],
[pattern matching][], [the pipeline operator][], and more. This engine has also
been used to find bugs in ECMA-262 and test262, the test suite for
conforming JavaScript implementations.

## Requirements

To run engine262 itself, a engine with support for recent ECMAScript features
is needed. Some of the newer features we use are:

- Exponentiation operator `**` (ES2016)
- Trailing commas in function declarations and calls (ES2017)
- Object spread properties (ES2018)

Additionally, the CLI (`bin/engine262.mjs`) test262 runner (`test/test262.mjs`)
requires a recent version of Node.js.

----

engine262 strives to function as independently from the host JavaScript engine
as possible. However, we still depend on the correctness of the host for a
variety of tasks we deem too fundamental to the platform for us to implement.
Some of them are:

- Mathematical operations (including most arithmetic operators and most methods
  on the `Math` global object)
- Reasonably sound implementation of "regular" objects, `Map` and `Set`
  objects, `TypedArray` objects, etc.

Some other operations also depend on the host's correctness at this moment,
though we hope to develop an independent implementation in the future:

- String-to-number conversion
- Regular expressions, when first implemented, will probably proxy the host's
  implementation

In particular, V8 versions between 6.7 and 7.0 (inclusive) had a bug that
resulted in the following:

```js
// Generate a new array with the spread syntax, but then discard the resulting
// array. This allows the following bug to occur.
[...[]];

// This should return false, as Object.is() treats positive and negative zeros
// as different, but returns true on these buggy hosts.
Object.is(-0, 0);
```

This bug is reproducible for all versions of Node.js newer than 10.3.0 (not
including 10.3.0). Many test262 tests that deal with negative zeros fail on
affected Node.js versions because of that. (See [nodejs/node#25221][].)

## Using engine262

First, install the required dependencies through `$ npm install` or `$ yarn`.

Then, build engine262 by running `$ npm run build` or `$ yarn build`.

> This process results in a build that does not use the do-expression feature
> in JavaScript (currently a Stage 1 proposal), which could be run in any host
> that satisfies the requirements above.
>
> However, it is somewhat less tested than a build that does use it, which
> could be generated by prepending the build command with
> `USE_DO_EXPRESSIONS=1`. On Node.js and V8, the resultant build requires
> `--harmony-do-expression` to execute.

After that, you can use the attached CLI program that functions as a REPL:

`$ bin/engine262.js`

Or, use the API:

```js
'use strict';

const { Realm, initializeAgent } = require('engine262');

initializeAgent({
  // onDebugger() {},
  // ensureCanCompileStrings() {},
  // promiseRejectionTracker() {},
  // hasSourceTextAvailable() {},
})

const realm = new Realm();

realm.evaluateScript(`
'use strict';

async function* numbers() {
  let i = 0;
  while (true) {
    const n = await Promise.resolve(i++);
    yield n;
  }
}

(async () => {
  for await (const item of numbers()) {
    print(item);
  }
})();
`);

// a stream of numbers fills your console. it fills you with determination.
```

## Related Projects

Many people and organizations have attempted to write a JavaScript interpreter
in JavaScript much like engine262, with different goals. Some of them are
included here for reference, though engine262 is not based on any of them.

- https://github.com/facebook/prepack
- https://github.com/mozilla/narcissus
- https://github.com/NeilFraser/JS-Interpreter
- https://github.com/metaes/metaes
- https://github.com/Siubaak/sval

[Babel]: https://babeljs.io/
[COC]: https://github.com/devsnek/engine262/blob/master/CODE_OF_CONDUCT.md
[do expressions]: https://github.com/tc39/proposal-do-expressions
[irc]: ircs://chat.freenode.net:6697/engine262
[irc-webchat]: https://webchat.freenode.net/?channels=engine262
[nodejs/node#25221]: https://github.com/nodejs/node/issues/25221
[optional chaining]: https://github.com/tc39/proposal-optional-chaining
[pattern matching]: https://github.com/tc39/proposal-pattern-matching
[the pipeline operator]: https://github.com/tc39/proposal-pipeline-operator
