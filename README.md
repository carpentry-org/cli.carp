# cli

A simple CLI library for Carp.

```clojure
(load "https://veitheller.de/git/carpentry/cli@0.0.6")

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
(load "https://veitheller.de/git/carpentry/cli@0.0.6")
```

## Usage

`CLI` should be built using combinators, as in the example above. It has, as of
now, three option types: integrals (longs), floating point numbers (doubles),
and strings. They can be built using `CLI.int`, `CLI.float`, and `CLI.str`,
respectively. Their structure is always the same:

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

Once you’re done building your flag structure, you can run `CLI.parse`. It
will not abort the program on error, instead it will tell you what went wrong
in a `Result.Error`. If it succeeds, the `Result.Success` contains a `Map` from
the long flag name to the value. The values are not in the map if they are
unset.

<hr/>

Have fun!
