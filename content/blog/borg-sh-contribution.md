+++
title = "Contributing a feature to BorgBackup"
date = 2023-05-30T10:36:23-07:00
draft = true
[taxonomies]
tags = ["Python", "shell", "regex", "parsing", "opensource"]
+++

# Finding an issue

I recently used
[Borg](https://borgbackup.readthedocs.io/en/stable/index.html) for the
first time to backup my Linux system. It\'s a fantastic tool, and with
[Vorta](https://vorta.borgbase.com/) as a GUI frontend it\'s very easy
to use. Both also happen to be written in Python (aside from some
performance-critical bits in C). So I looked over their GitHub
repositories for a good way to start contributing.

Having never contributed to an open-source Python project before, I
filtered by the \"good first issue\" label and found an ideal candidate:
[to add shell-style
alternatives](https://github.com/borgbackup/borg/issues/7602) to Borg\'s
pattern-matching functionality. Thomas Waldemann opened the issue and
pointed to a TODO comment in the `borg.helpers.shellpattern` module. I
am grateful for such an easy onramp to contribute.

So what was the issue? Borg can be given patterns by which to include
directories and files to backup. It accepts shell-style globs, which it
converts to regular expressions. The implementation of this conversion
is derived from Python\'s built-in [fnmatch
module](https://github.com/python/cpython/blob/67a8469237ebeee33733a5554ebfb4233e9752b8/Lib/fnmatch.py#L74),
but which is not an exhaustive solution. Borg\'s version of the
`translate` function, for instance, supports `**/...` patterns, which
include all directories (that\'s the double star). The issue requested
another supplementation: to convert shell-style brace expansion to
regular expression groups: `{alt1,alt2}` becomes `(alt1|alt2)`

## Setup: Distrobox to the rescue

Borg\'s documentation includes a [contributor\'s
guide](https://borgbackup.readthedocs.io/en/latest/development.html#),
which
[links](https://borgbackup.readthedocs.io/en/latest/installation.html#git-installation)
to using git to install from source, which in turn has
[instructions](https://borgbackup.readthedocs.io/en/latest/installation.html#source-install)
for installing dependencies. It\'s worth mentioning this stuff because I
ran into a problem I could not resolve: tests failed to run after
checking out my fork of Borg.

At first, I installed from the main borgbackup/borg.git repository,
because at this point I wasn\'t sure I would try to contribute. Setup
went well enough. But upon forking to my GitHub account and repeating
the same steps to install and setup my development environment, the
testing suite failed to run. The error message said something about the
tag identifying the borg installation being incorrect. Since I was using
separate virtual environments, I can\'t see how one messed with the
other.

Whatever the issue, I circumvented it by setting up a
[distrobox](https://distrobox.privatedns.org/):

```sh
distrobox create --image quay.io/fedora/fedora:38 --name fedora-38-box distrobox enter fedora-38-box
```

From there, I could install dependencies in a containerized environment.

# Getting to Work

## Exploration

Since it had been a while since I last worked with globs and regex of
much complexity, I opened a new `.py` file and a terminal and started
playing around. My plan was first to figure out how to translate valid
shell-style alternatives (the braces and commas) to regular expressions
(parentheses and pipes). Then I could work to integrate that logic into
the existing function.

Some considerations:

- the braces need not to be escaped, i.e. they cannot be preceded by
  an odd number of backslashes
- the inner content needs to include at least one comma, otherwise
  bash treats the braces as regular characters
- alternatives can be nested as well as sequenced
- patterns can include non-escaped, un-paired braces

### Trying regex

Python\'s `re` module was already imported by borg\'s `shellpattern`
module, so I wanted to give that a try. If I could match paired braces,
then I could use `re.match` to get their indices in the pattern string.
The closest I got was a pattern that could manage un-paired braces and,
with a bit more work, nested pairs:

```python
re.compile(
    """[^\\(\\\\)*]?  # exclude odd number of backslashes, if present
{                 # find an opening brace
.*                # match anything
[^\\(\\\\)*]?     # exclude odd number of backslashes, if present
}                 # find a closing brace
""", re.VERBOSE)
```

The problem is that it misses sequences of alternatives. For example
`"{foo,bar}{bar,baz}"` would translate, incorrectly, to
`"(foo|bar}{bar|baz)"` instead of, correctly, to `"(foo|bar)(bar|baz)"`.

## Parsing left to right

This particular translation problem was very similar to a basic
operation in computer programming: parsers. A parser traverses the text,
identifying language-valid tokens and constructing a syntax tree. Blocks
of code are often grouped using braces and/or parentheses. A stack can
be used to keep track of where the parser is relative to such paired
symbols. When an open brace is read, it gets added to a stack; when a
closing brace is read, the most recent open brace is popped from the
stack.

At this point, I also suspected that I would need the index of every
valid brace so that the characters they contained could be handled as
shell-style alternatives/regex groups. To track this, I mapped the index
of the opening brace to that of its closing counterpart. Python\'s
built-in dict makes this trivial. When a new opening brace is found, add
it as a key to the dict and set its value to some default that could not
ever map to a valid closing brace. I chose -1 (later I would change this
to `None`).

Here\'s the parsing function:

```python
from queue import LifoQueue


def parse_braces(pat: str) -> dict[int, int]:
    """Returns the index values of paired curly braces in `pat` as a dict mapping."""
    curly_q = LifoQueue()
    pairs: dict[int, int] = dict()

    for idx, c in enumerate(pat):
        if c == "{":
            if idx == 0 or pat[idx - 1] != "\\":
                # Opening brace is not escaped.
                # Add to dict
                pairs[idx] = -1
                # Add to queue
                curly_q.put(idx)
        if c == "}" and curly_q.qsize():
            # If queue is empty, then cannot close pair.
            if idx > 0 and pat[idx - 1] != "\\":
                # Closing brace is not escaped.
                # Pop off the index of the corresponding opening brace, which
                # provides the key in the dict of pairs, and set its value.
                pairs[curly_q.get()] = idx
    return pairs
```

Ultimately, after Thomas Waldmann\'s code review, this would return all valid index
pairs as a list of tuples.

# Integrating into Borg

With the braces-parsing function working, the next step was to integrate
that information into the actual translation function. I considered two
approaches:

1.  call a translation function to handle content within alternative
    groups
2.  convert the parsed braces and commas to parentheses and pipes before
    running the existing translation logic

While I did begin to implement the first, it required enough changes to
the original translation function that I opted to shift to the second
approach. The idea with the first approach was to separate off the
existing translation logic into its own helper function, which could be
called on slices of the input pattern. (Another idea I toyed around with
as part of this approach would have been to integrate the conversion of
braces and commas and to recursively call the translation function
within the alternatives/groups.)

By contrast, the second approach would integrate the added support
mostly by running the pattern through a pair of helper functions so that
all valid braces and commas would be translated prior to the main
translation logic. This meant chaining the parsing function with a
special translation function:

```python
def _translate_alternatives(pat: str) -> str:
    """Translates a shell-style pattern to a regular expression."""
    # Parse pattern for paired braces.
    # These will be converted to regex groups: {alt1,alt2} -> (alt1|alt2)
    brace_pairs = parse_braces(pat)

    pat_list = list(pat)  # Convert to list in order to subscript characters.

    # Convert non-escaped commas within groups to pipes.
    # Convert paired braces into parentheses, but only if at least one comma is present.
    # Passing, e.g. "{a\,b}.txt" to the shell expands to "{a,b}.txt", whereas
    # "{a\,,b}.txt" expands to "a,.txt" and "b.txt"
    for opening, closing in brace_pairs.items():
        commas = 0
        if val == -1:
            # Skip unpaired opening braces.
            continue

        for i in range(opening, closing + 1):
            if pat_list[i] == ",":
                if i == opening or pat_list[i - 1] != "\\":
                    pat_list[i] = "|"
                    commas += 1

        if commas > 0:  # problem here waiting to be discovered
            pat_list[opening] = "("
            pat_list[closing] = ")"

    return "".join(pat_list)
```

(Note that this still isn\'t
exactly right. See the section on Testing below.) The outer for loop
goes through every pair of braces, while the inner for loop checks for
non-escaped commas. When it finds one, it does two things: it converts
it to a pipe and it increments a counter. Remember, in order for shell
alternatives to work as expanded alternatives, there must be at least
one comma. Otherwise those braces remain as they are. So only when at
least one comma has been found will the braces be changed to
parentheses.

The only other change required was to add another conditional check
within Borg\'s existing translate function that would leave the
parnetheses and pipes alone:

```python
# borg.helpers.shellpattern
def translate(pat):
    pat = _translate_alternatives(pat)
    # ...
    n = len(pat)
    i = 0
    res = ""

    while i < n:
        # ...
        elif c in "(|)":
            if i > 0 and pat[i - 1] != "\\":
                res += c
```

This comes as the penultimate check within a while loop
that iterates over the pattern, right before an else clause that adds
the character escaped for regex: `res += re.escape(c)`

# Testing

Borg uses tox and pytest, which which makes testing a breeze. While I
was integrating my parsing and translation logic into `shellpattern.py`,
I was also running through some added tests.

While testing for nested groups, I realized that my initial translation
function did not take into account that _all_ commas within a parent
group, including those within a nested group, would be converted to
pipes. Why was this a problem? Because the comma counter would remain at
0 and therefore the nested braces would not get converted to
parentheses. To fix this, I separated out the counter logic and had it
check for pipes instead:

```python
for i in range(opening + 1, closing):  # Convert non-escaped commas to pipes.
    if pat_list[i] == ",":
        if i == opening or pat_list[i - 1] != "\\":
            pat_list[i] = "|"
            commas += 1
    elif pat_list[i] == "|" and (i == opening or pat_list[i - 1] != "\\"):
        # Nested groups have their commas converted to pipes when traversing the parent group.
        # So in order to confirm the presence of a comma in the original, shell-style pattern,
        # we must also check for a pipe.
        commas += 1
```

# Code Review

Thomas Waldmann responded with great feedback the following morning. He
really did make this experience fantastic for a new contributor like me.
Some key points:

## zsh != bash

He asked about a test I had come up with. Well, I had been coming up
with shell alternatives in zsh rather than bash. From within the
\@pytest.mark.parametrize fixture, I added a test that would confirm the
pattern (on the right-hand side) would match the string (on the
left-hand side):

`("bar/foobar", ["**/foo{ba[!z]*,[0-9]}"])`

The idea behind this is a directory structure containing at least ./bar/foobar.txt and
./bar/foobaz.txt. The pattern matches the former but ignores the latter.
In zsh, the exclamation mark used to negate the \"z\" is, like regex, a
caret \"\^\".

## returning tuples instead of a dict

Since the dict in the parsing function is used to iterate over the
paired index integers, Waldmann suggested I just return them as such.
Furthermore, since my specialized translation function was having to
check for unmatched braces (`if val == -1`), I could exclude those
mappings from the return statement and then remove that conditional
logic from the translation function:

```python
return [(opening, closing) for opening, closing in pairs.items() if closing is not None]
```

Note the conditional: I also took up Waldmann\'s
suggestion that `None` would be an even clearer representation of an
unmatched opening brace. Additionally, it\'s good practice to specify
that a reference `is not None` rather than leaving it as `if closing`,
since if the referenced object has a `__len__` method, its \"truthy\"
value could be unexpectedly True! Sure, in this case `closing` should
only ever be either `None` or a positive integer. But I appreciate the
clarity and readability.

# Merged

And here\'s the [merge
commit](https://github.com/borgbackup/borg/commit/021c9b656c2e081e1a8bc1e7b5ecda874b7a7b4a).

This is part of Borg 2, which is in alpha. So once it releases, you\'ll
be able to specify directories and files using shell-style alternatives
\: )
