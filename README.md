# cli

A simple CLI library for Carp.

```clojure
(load "git@github.com:carpentry-org/cli.carp@0.1.0")

(defn main []
  (let [p (=> (CLI.new @"My super cool tool!")
              (CLI.add &(CLI.int "flag" "f" "my flag" true))
              (CLI.add &(CLI.str "thing" "t" "my thing" false @"hi" &[@"a" @"b" @"hi"])))]
    (match (CLI.parse &p)
      (Result.Success flags)
        (println* &(str &(Map.get &flags "flag")) " " &(str &(Map.get &flags "thing")))
      (Result.Error msg) (do (IO.errorln &msg) (CLI.usage &p)))))
```

## Installation

```clojure
(load "git@github.com:carpentry-org/cli.carp@0.1.0")
```

## Usage

`CLI` should be built using combinators, as in the example above. It has, as of
now, three option types: integrals (longs), floating point numbers (doubles),
and strings. They can be built using `CLI.int`, `CLI.float`, `CLI.bool`, and
`CLI.str`, respectively. Their structure is always the same, except for
booleans:

```clojure
(CLI.int <long> <short> <description> <required?>)
; or
(CLI.int <long> <short> <description> <required?> <default>)
; or
(CLI.int <long> <short> <description> <required?> <default> <options-array>)
```

You’ll have to set a default if you want to specify options, although you can
set it to `(Maybe.Nothing)` if you want to make sure that it has to be set
manually.

Booleans neither take defaults nor options. If a boolean flag receives a value,
it will be read as true unless it’s the string `false`.

### Positional arguments

Positional arguments are non-flag tokens matched by position. Build them with
`CLI.pos-str`, `CLI.pos-int`, or `CLI.pos-float`:

```clojure
(CLI.pos-str <name> <description> <required?>)
```

Add them to the parser with `CLI.add-pos`. Flags and positionals can be
interleaved freely on the command line.

Once you’re done building your flag structure, you can run `CLI.parse`. It
will not abort the program on error, instead it will tell you what went wrong
in a `Result.Error`. If it succeeds, the `Result.Success` contains a `Map` from
the long flag name (or positional argument name) to the value. The values are
not in the map if they are unset.

### Subcommands

Bigger tools tend to be shaped like `git commit` / `git push` or `docker run` /
`docker build`: a program name followed by a subcommand, where each subcommand
is an independent parser with its own description, options, and positionals.
`CLI.App` gives you exactly that, layered on top of the `Parser` you already
know. It is purely additive — a plain `Parser` still works exactly as before.

Build each subcommand as a normal `Parser`, then register it on an `App` under a
name with `CLI.App.add`:

```clojure
(defn main []
  (let [commit (=> (CLI.new @"record changes to the repository")
                   (CLI.add &(CLI.str "message" "m" "commit message" true)))
        push   (=> (CLI.new @"update remote refs")
                   (CLI.add &(CLI.bool "force" "f" "force the push"))
                   (CLI.add-pos &(CLI.pos-str "remote" "the remote to push to" false)))
        app    (=> (CLI.App.new @"a tiny git")
                   (CLI.App.add "commit" &commit)
                   (CLI.App.add "push" &push))]
    (match (CLI.App.parse &app)
      (Result.Success chosen)
        (println* "ran " (Pair.a &chosen) " with "
                  &(Map.length (Pair.b &chosen)) " values")
      (Result.Error msg)
        (if (empty? &msg) (CLI.App.usage &app) (IO.errorln &msg)))))
```

`CLI.App.parse` (and its explicit-array sibling `CLI.App.parse-from`) reads the
first token to pick the subcommand and hands the remaining tokens to that
subcommand’s parser. On success it returns a `Pair` of the chosen subcommand
name and its parsed value `Map` — so you learn both *which* command ran and
*what* it was given. On error it returns a message: an empty one means `--help`
was requested (just like the flat parser), otherwise it describes a missing
subcommand, an unknown subcommand, or the subcommand’s own parse error.

`CLI.App.usage` lists the registered subcommands with their descriptions, and
`CLI.App.usage-for` prints the detailed usage of a single subcommand.

<hr/>

Have fun!
