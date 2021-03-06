# Infinite backtracking problem

Some regular expressions are looking simple, but can execute veeeeeery long time, and even "hang" the JavaScript engine.

Sooner or later all developers occasionally meets this behavior.

The typical situation -- a regular expression works fine for some time, and then starts to "hang" the script and make it consume 100% of CPU.

That may even be a vulnerability. For instance, if JavaScript is on the server and uses regular expressions on user data. There were many vulnerabilities of that kind even in widely distributed systems.

So the problem is definitely worth to deal with.

[cut]

## Example

The plan will be like this:

1. First we see the problem how it may occur.
2. Then we simplify the situation and see why it occurs.
3. Then we fix it.

For instance let's consider searching tags in HTML.

We want to find all tags, with or without attributes -- like `subject:<a href="..." class="doc" ...>`. We need the regexp to work reliably, because HTML comes from the internet and can be messy.

In particular, we need it to match tags like `<a test="<>" href="#">` -- with `<` and `>` in attributes. That's allowed by [HTML standard](https://html.spec.whatwg.org/multipage/syntax.html#syntax-attributes).

Now we can see that a simple regexp like `pattern:<[^>]+>` doesn't work, because it stops at the first `>`, and we need to ignore `<>` inside an attribute.

```js run
// the match doesn't reach the end of the tag - wrong!
alert( '<a test="<>" href="#">'.match(/<[^>]+>/) ); // <a test="<>
```

We need the whole tag.

To correctly handle such situations we need a more complex regular expression. It will have the form  `pattern:<tag (key=value)*>`.

In the regexp language that is: `pattern:<\w+(\s*\w+=(\w+|"[^"]*")\s*)*>`:

1. `pattern:<\w+` -- is the tag start,
2. `pattern:(\s*\w+=(\w+|"[^"]*")\s*)*` -- is an arbitrary number of pairs `word=value`, where the value can be either a word `pattern:\w+` or a quoted string `pattern:"[^"]*"`.

That doesn't yet support the details of HTML grammer, for instance strings can be in 'single' quotes, but these can be added later, so that's somewhat close to real life. For now we want the regexp to be simple.

Let's try it in action:

```js run
let reg = /<\w+(\s*\w+=(\w+|"[^"]*")\s*)*>/g;

let str='...<a test="<>" href="#">... <b>...';

alert( str.match(reg) ); // <a test="<>" href="#">, <b>
```

Great, it works! It found both the long tag `match:<a test="<>" href="#">` and the short one `match:<b>`.

Now let's see the problem.

If you run the example below, it may hang the browser (or another JavaScript engine):

```js run
let reg = /<\w+(\s*\w+=(\w+|"[^"]*")\s*)*>/g;

let str = `<tag a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  
  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b`;

*!*
// The search will take a long long time
alert( str.match(reg) );
*/!*
```

Some regexp engines can handle that search, but most of them don't.

What's the matter? Why a simple regular expression on such a small string "hangs"?

Let's simplify the situation by removing the tag and quoted strings, we'll look only for attributes:

```js run
// only search for space-delimited attributes
let reg = /<(\s*\w+=\w+\s*)*>/g;

let str = `<a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b
  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b`;

*!*
// the search will take a long, long time
alert( str.match(reg) );
*/!*
```

The same.

Here we end the demo of the problem and start looking into what's going on.

## Backtracking

To make an example even simpler, let's consider `pattern:(\d+)*$`.

In most regexp engines that search takes a very long time (careful -- can hang):

```js run
alert( '12345678901234567890123456789123456789z'.match(/(\d+)*$/) );
```

So what's wrong with the regexp?

Actually, it looks a little bit strange. The quantifier `pattern:*` looks extraneous. If we want a number, we can use `pattern:\d+$`.

Yes, the regexp is artificial, but the reason why it is slow is the same as those we saw above. So let's understand it.

What happen during the search of `pattern:(\d+)*$` in the line `subject:123456789z`?

1. First, the regexp engine tries to find a number `pattern:\d+`. The plus `pattern:+` is greedy by default, so it consumes all digits:

    ```
    \d+.......
    (123456789)z
    ```
2. Then it tries to apply the start around the parentheses `pattern:(\d+)*`, but there are no more digits, so it the star doesn't give anything.

    Then the pattern has the string end anchor `pattern:$`, and in the text we have `subject:z`.

    ```
               X
    \d+........$
    (123456789)z
    ```

    No match!
3. There's no match, so the greedy quantifier `pattern:+` decreases the count of repetitions (backtracks).

    Now `\d+` is not all digits, but all except the last one:
    ```
    \d+.......
    (12345678)9z
    ```
4. Now the engine tries to continue the search from the new position (`9`).

    The start `pattern:(\d+)*` can now be applied -- it gives the number `match:9`:

    ```

    \d+.......\d+
    (12345678)(9)z
    ```

    The engine tries to match `$` again, but fails, because meets `subject:z`:

    ```
                 X
    \d+.......\d+
    (12345678)(9)z
    ```

    There's no match, so the engine will continue backtracking.
5. Now the first number `pattern:\d+` will have 7 digits, and the rest of the string `subject:89` becomes the second `pattern:\d+`:

    ```
                 X
    \d+......\d+
    (1234567)(89)z
    ```

    ...Still no match for `pattern:$`.

    The search engine backtracks again. Backtracking generally works like this: the last greedy quantifier decreases the number of repetitions until it can. Then the previous greedy quantifier decreases, and so on. In our case the last greedy quantifier is the second `pattern:\d+`, from `subject:89` to `subject:8`, and then the star takes `subject:9`:

    ```
                   X
    \d+......\d+\d+
    (1234567)(8)(9)z
    ```
6. ...Fail again. The second and third `pattern:\d+` backtracked to the end, so the first quantifier shortens the match to `subject:123456`, and the star takes the rest:

    ```
                 X
    \d+.......\d+
    (123456)(789)z
    ```

    Again no match. The process repeats: the last greedy quantifier releases one character (`9`):

    ```
                   X
    \d+.....\d+ \d+
    (123456)(78)(9)z
    ```
7. ...And so on.

The regular expression engine goes through all combinations of `123456789` and their subsequences. There are a lot of them, that's why it takes so long.

A smart guy can say here: "Backtracking? Let's turn on the lazy mode -- and no more backtracking!".

Let's replace `pattern:\d+` with `pattern:\d+?` and see if it works (careful, can hang the browser)

```js run
// sloooooowwwwww
alert( '12345678901234567890123456789123456789z'.match(/(\d+?)*$/) );
```

No, it doesn't.

Lazy quantifiers actually do the same, but in the reverse order. Just think about how the search engine would work in this case.

Some regular expression engines have tricky built-in checks to detect infinite backtracking or other means to work around them, but there's no universal solution.

In the example above, when we search `pattern:<(\s*\w+=\w+\s*)*>` in the string `subject:<a=b  a=b  a=b  a=b` -- the similar thing happens.

The string has no `>` at the end, so the match is impossible, but the regexp engine does not know about it. The search backtracks trying different combinations of `pattern:(\s*\w+=\w+\s*)`:

```
(a=b a=b a=b) (a=b)
(a=b a=b) (a=b a=b)
...
```

## How to fix?

The problem -- too many variants in backtracking even if we don't need them.

For instance, in the pattern `pattern:(\d+)*$` we (people) can easily see that `pattern:(\d+)` does not need to backtrack.

Decreasing the count of `pattern:\d+` can not help to find a match, there's no matter between these two:

```
\d+........
(123456789)z

\d+...\d+....
(1234)(56789)z
```

Let's get back to more real-life example: `pattern:<(\s*\w+=\w+\s*)*>`. We want it to find pairs `name=value` (as many as it can). There's no need in backtracking here.

In other words, if it found many `name=value` pairs and then can't find `>`, then there's no need to decrease the count of repetitions. Even if we match one pair less, it won't give us the closing `>`:

Modern regexp engines support so-called "possessive" quantifiers for that. They are like greedy, but don't backtrack at all. Pretty simple, they capture whatever they can, and the search continues. There's also another tool called "atomic groups" that forbid backtracking inside parentheses.

Unfortunately, but both these features are not supported by JavaScript.

Although we can get a similar affect using lookahead. There's more about the relation between possessive quantifiers and lookahead in articles [Regex: Emulate Atomic Grouping (and Possessive Quantifiers) with LookAhead](http://instanceof.me/post/52245507631/regex-emulate-atomic-grouping-with-lookahead) and [Mimicking Atomic Groups](http://blog.stevenlevithan.com/archives/mimic-atomic-groups).

The pattern to take as much repetitions as possible without backtracking is: `pattern:(?=(a+))\1`.

In other words, the lookahead `pattern:?=` looks for the maximal count `pattern:a+` from the current position. And then they are "consumed into the result" by the backreference `pattern:\1`.

There will be no backtracking, because lookahead does not backtrack. If it found like 5 times of `pattern:a+` and the further match failed, then it doesn't go back to 4.

Let's fix the regexp for a tag with attributes from the beginning of the chapter`pattern:<\w+(\s*\w+=(\w+|"[^"]*")\s*)*>`. We'll use lookahead to prevent backtracking of `name=value` pairs:

```js run
// regexp to search name=value
let attrReg = /(\s*\w+=(\w+|"[^"]*")\s*)/

// use it inside the regexp for tag
let reg = new RegExp('<\\w+(?=(' + attrReg.source + '*))\\1>', 'g');

let good = '...<a test="<>" href="#">... <b>...';

let bad = `<tag a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b
  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b  a=b`;

alert( good.match(reg) ); // <a test="<>" href="#">, <b>
alert( bad.match(reg) ); // null (no results, fast!)
```

Great, it works! We found a long tag  `match:<a test="<>" href="#">` and a small one `match:<b>` and didn't hang the engine.

Please note the `attrReg.source` property. `RegExp` objects provide access to their source string in it. That's convenient when we want to insert one regexp into another.
