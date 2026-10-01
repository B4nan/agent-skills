# Prototype pollution

Reports in this family use `__proto__`, `constructor`, or `prototype` as object keys. Most are inert; a few are severe. The difference is mechanical and you can test it.

## Inert vs consequential

- **Inert:** reassigning *a throwaway object's* prototype. `copy.__proto__ = x` (or `Object.assign(copy, parsed)` where `parsed` has an own `__proto__` key) changes what `copy` inherits from. If `copy` is discarded or only read back by the same code, nothing outside it changes.
- **Consequential:** writing *through* a prototype onto a shared object. `obj.__proto__.foo = x` is `Object.prototype.foo = x`, which every object in the process now inherits. Same for `obj.constructor.prototype.foo = x`. Recursive merge and deep-set helpers are the usual route, because they walk into `__proto__` as if it were a nested object.

Always check which one the code actually does: run it, then probe a fresh object (`({}).foo`, `Object.prototype.hasOwnProperty('foo')`). Report the literal output.

## Where the keys come from

`JSON.parse` creates `__proto__` as an ordinary **own** property; it does not set the prototype. The key only becomes dangerous when later code assigns it (`target[key] = value`, `Object.assign`, recursive merge). Object spread does not: it copies `__proto__` back as an own key without invoking the setter. Object literals in source code (`{ __proto__: x }`) behave differently and are not the attacker's route.

## Related lookup bug

`key in lookupObject` and `lookupObject[key]` return inherited members for `constructor`, `toString`, `__proto__` and friends, so user-supplied keys can be misclassified as known commands, handlers, or options. Usually this produces a crash or a wrong branch rather than pollution. Fix with `Object.hasOwn(lookup, key)` (`Object.prototype.hasOwnProperty.call(lookup, key)` before Node 16.9) or a null-prototype map (`Object.create(null)` / `Map`).

## Narrowest fix

Guard the specific key at the specific assignment or merge point (skip or reject `__proto__` / `constructor` / `prototype` there), or copy into a null-prototype object. Do not strip those keys everywhere: blanket guards silently drop data that was handled correctly, and an existing test usually documents that.
