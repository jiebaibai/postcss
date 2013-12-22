# PostCSS

PostCSS is a framework for CSS postprocessors,
to modify CSS by your JS function.

It takes care of most common CSS tool tasks:

1. parses CSS;
2. gives you usable JS API to edit CSS node tree;
3. dumps modified node tree into CSS string;
4. generates (or modifies existent) source map for your changes;

You can use this framework to write you own:

* CSS minifier or beautifizer.
* Grunt plugin to generate sprites, include `data-uri` images
  or any other works.
* Text editor plugin to automate CSS routine.
* Command-line CSS tool.

Sponsored by [Evil Martians](http://evilmartians.com/).

## Build with PostCSS

* [Autoprefixer](https://github.com/ai/autoprefixer)
* [grunt-pixrem](https://github.com/robwierzbowski/grunt-pixrem)

## Quick Example

Let’s fix forgotten `content` property in `::before` and `::after`:

```js
var postcss = require('postcss');

var contenter = postcss(function (css) {
    css.eachRule(function (rule) {
        if ( rule.selector.match(/::(before|after)/) ) {
            // In every ::before/::after rule

            // Did we forget content property?
            var good = rule.some(function (i) { return i.prop == 'content'; });

            if ( !good ) {
                // Add content: '' if we forget it
                rule.prepend({ prop: 'content', value: '""' });
            }

        }
    });
});
```

And then CSS with forgotten `content`:

```css
a::before {
    width: 10px;
    height: 10px
}
```

will be fixed by our new `contenter`:

```js
var fixed = contenter.process(css).css;
```

to:

```css
a::before {
    content: "";
    width: 10px;
    height: 10px
}
```

## Features

### Source Map

PostCSS generates source map for its transformations:

```js
result = processor.process(css, { map: true, from: 'from.css', to: 'to.css' });
result.css // String with processed CSS
result.map // Source map
```

And modifies source map from previous step (like Sass preprocessor):

```js
var sassMap = fs.readFileSync('from.sass.css.map');
processor.process(css, { map: sassMap, from: 'from.sass.css', to: 'to.css' });
```

### Preserves code formatting and indentations

PostCSS will not change any byte of a rule if you don’t modify its node:

```js
postcss(function (css) { }).process(css).css == css;
```

And when you modify CSS nodes, PostCSS will try to copy coding style:

```js
contenter.process("a::before{color: black}")
// a::before{content: '';color: black}

contenter.process("a::before {\n  color: black;\n  }")
// a::before {
//   content: '';
//   color: black;
//   }
```

## Why PostCSS Better Than …

### Preprocessors

Preprocessors (like Sass or Stylus) give us special language with variables,
mixins, statements and compile it to CSS. Compass, nib and other mixins
libraries use these languages to work with prefixes, sprites and inline images.

But Sass and Stylus languages were created to be syntax-sugar for CSS.
Writing really complicated programs using preporcessor languages
is very difficult. [Autoprefixer] is absolutely impossible to implement
on top of Sass.

PostCSS gives you comfort and power of JS or CoffeeScript to working with CSS.
You can do really magic things with wide range of [npm] libraries.

But postprocessors are not enemies for preprocessors. Sass and Stylus are still
the best way to improve readability and add some syntax sugar to CSS.
You can easily combine preprocessors and postprocessors.

[Autoprefixer]: https://github.com/ai/autoprefixer
[npm]:          https://npmjs.org/

### RegExp

Some Grunt plugins modify CSS with regular expressions. But CSS parser and
node tree are much safer way to edit CSS. Also regexps will break source maps
generated by preprocessors.

### CSS Parsers

There are a lot of good CSS parsers, like [Gonzales]. But they help you only
with first step.

Unlike them PostCSS gives you useful high level API (for example,
safe iterators) and changes source map generator (or modifier for existing
source map from preprocessors).

[Gonzales]: https://github.com/css/gonzales

### Rework

[Rework] was a first CSS postprocessors framework. PostCSS is very similar
to it.

But Rework right now doesn’t support source maps.

Also Rework has no high level API and doesn’t preserve formatting
and indentations while transforming your CSS. Thus it can’t be used
to implement text editor plugins.

Unlike it PostCSS preserves all spaces and code formatting.
If you don’t change rule, output will be byte‑to‑byte equal.

[Rework]: https://github.com/visionmedia/rework

## Usage

### Processor

Function `postcss(fn)` creates a processor from your function:

```js
var postcss = require('postcss');

var processor = postcss(function (css) {
    // Code to modify CSS
});
```

If you want to combine multiple processors (and parse CSS only once),
you can create an empty processor and add several functions
using `use(fn)` method:

```js
var all = postcss().
          use(prefixer).
          use(minifing);
```

Processor function can change just the current CSS node tree:

```js
postcss(function (css) {
    css.append( /* new rule */ )
});
```

or create a completely new CSS root node and return it instead:

```js
postcss(function (css) {
    var newCSS = postcss.root()
    // Add rules and declarations
    return newCSS;
});
```

Processor transforms some CSS using `process(css, opts)` method:

```js
var doubler = postcss(function (css) {
    // Clone each declaration
    css.eachDecl(function (decl) {
        decl.parent.prepend( decl.clone() );
    });
});

var css    = "a { color: black; }";
var result = doubler.process(css);

result.css //=> "a { color: black; color: black; }"
```

You can set original CSS filename via `from` option and make syntax error
messages much more helpful:

```js
var wrong = "a {";
processor.process(wrong, { from: 'main.css' });
//=> Can't parse CSS: Unclosed block at line 1:1 in main.css
```
### Source Map

PostCSS generates source map, if you set `map` option to `true`
in `process(css, opts)` method.

You must set input and output CSS files paths (using `from` and `to`
options respectively) to generate correct source map.

```js
var result = processor.process(css, {
    map:  true,
    from: 'main.css',
    to:   'main.out.css'
});

result.map //=> '{"version":3,"file":"main.out.css","sources":["main.css"],"names":[],"mappings":"AAAA,KAAI"}'

fs.writeFileSync('main.out.css.map', result.map);
```

PostCSS can also modify previous source map (for example, from Sass
compilation). So if you compile: Sass to CSS and then minify CSS
by postprocessor, final source map will contain mapping from Sass code
to minified CSS.

Just set `map` option to an original source map (a string or a JS object):

```js
var result = minifier.process(css, {
    map:   fs.readFileSync('main.sass.css.map'),
    from: 'main.sass.css',
    to:   'main.min.css'
});

result.map //=> Source map from main.sass to main.min.css
```

### Nodes

Processor function receives `Root` node with CSS node tree inside.

```js
var processor = postcss(function (cssRoot) {
});
```

There are 3 types of child nodes: `Comment`, `AtRule`, `Rule` and `Declaration`.
All nodes have `toString()` and `clone()` methods.

You can parse CSS and get a `Root` node by `postcss.parse(css, opts)` method:

```js
var postcss = require('postcss');

var cssRoot = postcss.parse('a { }');
```

### Node Source

Every node stores its origin file (if you set `from` option to `process`
or `parse` method) and position at `source` property:

```js
var root = postcss.parse(css, { from: 'main.css' });
var rule = root.rules[1];

rule.source.file  //=> 'main.css'
rule.source.start //=> { line: 5,  position: 1 }
rule.source.end   //=> { line: 10, position: 5 }
```

### Whitespaces

All nodes (exclude `Root`) have `before` property with indentation
and all earlier spaces.

Nodes with children (`Root`, `AtRule` and `Rule`) contain also `after` property
with spaces after last child and before `}` or end of file.

```js
var root = postcss.parse("a {\n  color: black;\n}\n");

root.after                    //=> "\n" from end of file
root.rules[0].after           //=> "\n" before }
root.rules[0].decls[0].before //=> "\n  " before color: black
```

The simplest way to minify CSS is to set `before` and `after` properties
to an empty string:

```js
var minifier = postcss(function (css) {
    css.eachDecl(function (decl) {
        decl.before = '';
    });
    css.eachRule(function (rule) {
        rule.before = '';
        rule.after  = '';
    });
    css.eachAtRule(function (atRule) {
        atRule.before = '';
        atRule.after  = '';
    });
});

var css = "a{\n  color:black\n}\n";
minifier.process(css).css //=> "a{color:black}"
```

### Raw Properties

Some CSS values (selectors, comment content, at-rule params
and declaration values) can contain trailing spaces and comments.
PostCSS will clean them for you:

```js
var root = postcss.parse("a /**/ b {}");
var ab   = root.rules[0];

ab.selector      //=> 'a  b' trimmed and cleaned from comments
ab._selector.raw //=> 'a /**/ b ' original raw value
```

But PostCSS saves raw content to stringify it to CSS, if you don’t
set new value. As you can remember, PostCSS tries to save origin CSS
byte-to-byte, when it’s possible:

```js
ab.toString() //=> 'a /**/ b {}' with comment

ab.selector = '.link b';
ab.toString() //=> '.link b' you change value and magic was gone
```

### Containers

`Root`, `AtRule` and `Rule` nodes can contain children in `rules` or `decls`
property.

There are common method to work with children:

* `append(newChild)` to add child at the end of children list.
* `prepend(newChild)` to add child at the beginning of children list.
* `insertBefore(existsChild, newChild)` to insert new child before some
   existent child.
* `insertAfter(existsChild, newChild)` to insert new child after some
   existent child.
* `remove(existsChild)` to remove child.
* `index(existsChild)` to return child index.
* `some(fn)` to return true if `fn` returns true on any child.
* `every(fn)` to return true if `fn` returns true on all children.

Methods `insertBefore`, `insertAfter` and `remove` can receive child node
or child index as an `existsChild` argument. Have in mind that `index` works
much faster.

### Children

`Comment`, `AtRule`, `Rule` and `Declaration` nodes should be wrapped
in other nodes.

All children contain `parent` property with parent node:

```js
rule.decls[0].parent == rule;
```

All children has `removeSelf()` method:

```js
rule.decls[0].removeSelf();
```

But `remove(index)` in parent with child index is much faster:

```js
rule.each(function (decl, i) {
    rule.remove(i);
});
```

### Iterators

All parent nodes have `each` method to iterate over children nodes:

```js
root = postcss.parse('a { color: black; display: none }');

root.each(function (rule, i) {
    if ( rule.type == 'rule' ) {
        console.log(rule.selector, i); // Will log "a 0"
    }
});

root.rules[0].each(function (decl, i) {
    if ( rule.type != 'comment' ) {
        console.log(decl.prop, i); // Will log "color 0" and "display 1"
    }
});
```

Unlike `for {}`-cycle construct or `Array#forEach()` this iterator is safe.
You can mutate children while iteration and it will fix current index:

```js
rule.rules.forEach(function (decl, i) {
    rule.prepend( decl.clone() );
    // Will be infinity cycle, because on prepend current declaration become
    // second and next index will go to current declaration again
});

rule.each(function (decl, i) {
    rule.prepend( decl.clone() );
    // Will work correct (once clone each declaration), because after prepend
    // iterator index will be recalculated
});
```

Because CSS have nested structure, PostCSS contains recursive iterators
for different node types:

```js
root.eachDecl(function (decl, i) {
    // Each declaration inside root
});

root.eachRule(function (rule, i) {
    // Each rule inside root and any nested at-rules
});

root.eachAtRule(function (atRule, i) {
    // Each at-rule inside root and any nested at-rules
});
```

### Root Node

`Root` node contains entire CSS tree. Its children can be only `Comment`,
`AtRule` or `Rule` nodes in `rules` property.

You can create a new root using shortcut:

```js
var root = postcss.root();
```

Method `toString()` stringifies entire node tree to CSS string:

```js
root = postcss.parse(css);
root.toString() == css;
```

### Comment Node

```css
/* Block comment */
```

PostCSS create `Comment` nodes only for comments between rules or declarations.
Comments inside selectors, at-rules params, declaration values will be stored
in Raw property.

`Comment` has only one property: `content` with trimmed text inside comment.

```js
comment.content //=> "Block comment"
```

You can create a new comment using shortcut:

```js
var comment = postcss.comment({ content: 'New comment' });
```

### AtRule Node

```css
@charset 'utf-8';

@font-face {
    font-family: 'Cool'
}

@media print {
    img { display: none }
}
```

`AtRule` has two own properties: `name` and `params`.

As you see, some at-rules don’t contain any children (like `@charset`
or `@import`), some of at-rules can contain only declarations
(like `@font-face` or `@page`), but most of them can contain rules
and nested at-rules (like `@media`, `@keyframes` and others).

Parser selects `AtRule` content type by its name. If you create `AtRule`
node manually, it will detect own content type with new child type on first
`append` or other add method call:

```js
var atRule = postcss.atRule({ name: '-x-animations' });
atRule.rules        //=> undefined
atRule.decls        //=> undefined

atRule.append( postcss.rule({ selector: 'from' }) );
atRule.rules.length //=> 1
atRule.decls        //=> undefined
```

You can create a new at-rule using shortcut:

```js
var atRule = postcss.atRule({ name: 'charset', params: 'utf-8' });
```

### Rule Node

```css
a {
    color: black;
}
```

`Rule` node has `selector` property and contains `Declaration` and `Comment`children in `decls` property.

You can miss `Declaration` constructor in `append` and other insert methods:

```js
rule.append({ prop: 'color', value: 'black' });
```

Property `semicolon` indicates if last declaration in rule has semicolon or not:

```js
var root = postcss.parse('a { color: black }');
root.rules[0].semicolon //=> false

var root = postcss.parse('a { color: black; }');
root.rules[0].semicolon //=> true
```

You can create a new rule using shortcut:

```js
var rule = postcss.rule({ selector: 'a' });
```

### Declaration Node

```css
color: black
```

`Declaration` node has `prop` and `value` properties.

You can create a new declaration using shortcut:

```js
var decl = postcss.decl({ prop: 'color', value: 'black' });
```

Or use short form in `append()` and other add methods:

```js
rule.append({ prop: 'color', value: 'black' });
```
