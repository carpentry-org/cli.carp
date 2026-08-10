# cli

A simple CLI library for Carp.

```clojure
(load "git@github.com:carpentry-org/cli.carp@0.2.0")

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
(load "git@github.com:carpentry-org/cli.carp@0.2.0")
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

### Short flag bundling

Short options can be written the way you are used to typing them: `ls -la`,
`tar -xzf archive.tar.gz`, `grep -rn pattern`. A single-dash token that matches
no option exactly is decomposed into single-character short options, so `-av` is
`-a -v`.

Boolean options continue the bundle. The first option that takes a value ends
it and reads the rest of the token as its value, or the next token if there is
nothing left:

```
-n5        ; num = 5
-avn5      ; all, verbose, num = 5
-avn 5     ; all, verbose, num = 5
```

Only single-character short names take part in this. A token that names an
option exactly is always resolved as that option first, so a multi-character
short name like `th` keeps working, and so does a long name given with a single
dash. If any letter of a bundle is not a known short option, the whole token is
rejected — nothing in it is applied.

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
      (CLI.Dispatch.Parsed chosen)
        (println* "ran " (Pair.a &chosen) " with "
                  &(Map.length (Pair.b &chosen)) " values")
      (CLI.Dispatch.AppHelp)          (CLI.App.usage &app)
      (CLI.Dispatch.CommandHelp name) (CLI.App.usage-for &app &name)
      (CLI.Dispatch.Failure msg)      (IO.errorln &msg))))
```

`CLI.App.parse` (and its explicit-array sibling `CLI.App.parse-from`) reads the
first token to pick the subcommand and hands the remaining tokens to that
subcommand’s parser. Rather than a bare `Result`, it returns a `CLI.Dispatch`
that tells you exactly what happened, so you can respond with the *right* help:

- `Parsed` — success. Holds a `Pair` of the chosen subcommand name and its
  parsed value `Map`, so you learn both *which* command ran and *what* it was
  given.
- `AppHelp` — a top-level `--help`/`-h` was requested before any subcommand;
  show `CLI.App.usage`.
- `CommandHelp` — a subcommand’s *own* `--help`/`-h` was requested (e.g.
  `mytool commit --help`); it carries the subcommand’s name, so you can show
  `CLI.App.usage-for` for exactly that command instead of the whole app.
- `Failure` — carries an error message: a missing or unknown subcommand, the
  subcommand’s own parse error, or a leading option (see below).

Options come **after** the subcommand. A leading option other than `--help`/`-h`
— for example `mytool --verbose commit` — is a `Failure` with the message
`Expected a subcommand, got option: --verbose`; put such flags on the subcommand
instead (`mytool commit --verbose`).

`CLI.App.usage` lists the registered subcommands with their descriptions, and
`CLI.App.usage-for` prints the detailed usage of a single subcommand.

### Shell completion

The parser already knows every flag name, every declared value set, and — for
an `App` — every subcommand. `CLI.Completion` turns that into a completion
script, so the completions cannot drift from the flags your program actually
accepts. `bash` and `zsh` are supported:

```clojure
(defn main []
  (let [p (=> (CLI.new @"my super cool tool")
              (CLI.add &(CLI.bool "verbose" "v" "be verbose"))
              (CLI.add &(CLI.str "out" "o" "where to write" false))
              (CLI.add &(CLI.str "mode" "m" "how to run" false @"fast"
                                 &[@"fast" @"slow"])))]
    (IO.println &(CLI.Completion.bash &p "mytool"))))
```

That prints a script you can install the usual way — `mytool --completion bash
> /etc/bash_completion.d/mytool`, or straight into your shell with `source <(mytool
--completion bash)`. The zsh counterpart is `CLI.Completion.zsh`, and it works
both as an autoloaded `#compdef` file and sourced directly. The bash script
uses `mapfile`, so it needs bash 4 or newer.

With that installed, `mytool -<TAB>` offers `--verbose -v --out -o --mode -m
--help -h`, and `mytool --mode <TAB>` offers exactly `fast` and `slow`, because
that flag declared its value set.

For an `App`, use `CLI.Completion.App.bash` and `CLI.Completion.App.zsh`. They
complete the registered subcommand names first — with their descriptions, under
zsh — and once a subcommand has been chosen they continue with *that*
subcommand’s flags and value sets:

```clojure
(IO.println &(CLI.Completion.App.zsh &app "mytool"))
```

Where the library knows nothing more specific — the value of a flag that has no
declared value set, and every positional argument — the generated script falls
back to the shell’s own file name completion, which is what a user expects from
a command line tool. This is consistent across both shells.

Descriptions, flag names and declared values are free-form strings, so they are
quoted and escaped for the target shell. A description containing quotes,
brackets, backslashes or dollar signs cannot break the generated script, and a
declared value like `$HOME` or `` `id` `` is offered as that literal text rather
than being expanded or run. A declared value containing spaces stays a single
completion candidate, and one containing `*` or `?` is offered as itself rather
than as the file names it happens to match. (bash completion has nowhere to
show descriptions, so the bash script omits them.)

<hr/>

Have fun!
