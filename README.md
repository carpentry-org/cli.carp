# cli

A simple CLI library for Carp.

```clojure
(load "git@github.com:carpentry-org/cli.carp.git@master")

(defn main []
  (let [p (=> (CLI.new @"My super cool tool!")
              (CLI.add &(CLI.option "--flag" "-f" "my flag" true)))]
    (match (CLI.parse &p)
      (Result.Success p) (IO.println &(str &(CLI.get &p "--flag")))
      (Result.Error msg) (do (IO.errorln &msg) (CLI.usage &p)))))
```

## Usage

```clojure
(load "git@github.com:carpentry-org/cli.carp.git@master")
```

<hr/>

Have fun!
