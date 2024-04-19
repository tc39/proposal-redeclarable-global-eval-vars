# Make `eval`-introduced global `var`s redeclarable

Stage: 3

Author: Shu-yu Guo

Champion: Shu-yu Guo

## Motivation
Currently, at the global toplevel, it is an error to:

1. Redeclare a `var` or `function` binding with a `let` or `const` of the same name
1. Redeclare a non-configurable global property with a `let` or `const` of the same name
1. Redeclare a `let` or `const with a `let or `const` of the same name

While all global var and function bindings are also global properties, (1) is distinct from (2) because `var` and `function` bindings introduced by sloppy direct `eval` at the global toplevel are configurable (unlike `var` bindings introduced via script, which are non-configurable).

This distinction is specified by tracking all `var` binding names in the [[[VarNames]] field on the Global Environment Record](https://tc39.es/ecma262/#table-additional-fields-of-global-environment-records).

This proposal's claim is that this distinction has dubious utility and complicates both the mental model and implementation of the language.

First, the goal of preventing redeclaration is not met anyway. `var` bindings introduced by `eval` are already `delete`-able because they are configurable. So it's not like you can't redeclare them. You can, you just first have to delete the `var`.

While it is true that `eval`-introduced `var` bindings in function scopes are also `delete`-able, the key difference is that function scopes are _closed_ while the global toplevel scope is _open_. That is, contrast the following examples.

```html
<script>
eval("var x = 42");
</script>
<script>
delete x;  // Delete the eval-introduced `var`
</script>
<script>
let x;     // Redeclare x as a let
</script>
```

```javascript
function f() {
  // This is redeclaration error because the eval is evaluated
  // when there's already a let x in scope.
  //
  // That is, there's no way to actually delete-then-redeclare
  // an eval-introduced var with a lexical binding in a single
  // function scope.
  eval("var x = 42");
  delete x;
  let x;
}
```

Second, the [[VarNames]] list has to be tracked specially, and is basically an extra bit on the property descriptor for all properties on the global object. This is extra implementation complexity.

## Proposal

See https://github.com/tc39/ecma262/pull/3226 for the spec changes ([rendering](https://arai-a.github.io/ecma262-compare/?pr=3226)). That PR removes [[VarNames]], effectively reducing the 3 cases above to 2:

1. Redeclare a non-configurable global property with a `let` or `const` of the same name
1. Redeclare a `let` or `const` with a `let` or `const` of the same name

The alternative of making sloppy direct eval-introduced vars non-configurable is not considered owing to a much longer history, and its being less likely to be a web compatible change.

## Consequences

The main thing that changes is that the following snippet is now allowed. An `eval`-introduced `var` in the global toplevel scope can be redeclared by lexical bindings, effectively shadowing the configurable property on `globalThis`.

```html
<script>
eval("var x = 'var'");
</script>
<script>
let x = 'let';
console.log(x);            // 'let'
console.log(globalThis.x); // 'var'
</script>
```

It is web compatible to make this change as we are changing a throwing behavior to a non-throwing behavior.

## FAQ

### Does this break symmetry with function-scope `var`s and direct `eval`?

Not really, because function scopes and the global toplevel scope already behave very differently.

Per above, the global toplevel is an _open_ scope while function scopes are _closed_. This proposal changes observable behavior that is only observable by re-entering the global scope (e.g. via new `<script>` tags in the HTML embedding). In other words, if one were to use the global top-level scope like a closed scope and does not re-enter it, there is no observable difference.
