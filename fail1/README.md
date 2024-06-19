# `fail1`: a minimal bbin-compatible script

This folder reproduces a known limitation in `babashka/bbin` per 2024-06-19:

https://github.com/babashka/bbin/issues/78

## Observations

### Expected: this script can be installed.

### Actual: crash when we try to install.

``` shell
$ bbin-dev install .

Starting install...

bin    location
─────  ───────────────────────────────────────────────────────
fail1  file:///Users/teodorlu/dev/teodorlu/bbin-testdata/fail1

Install complete.

$ fail1
Error building classpath. Manifest file not found for local/deps in coordinate #:local{:root "/Users/teodorlu/dev/teodorlu/bbin-testdata/fail1"}
Exception in thread "main" clojure.lang.ExceptionInfo:  babashka.process.Process@591fb1b
	at babashka.process$check.invokeStatic(process.cljc:111)
	at babashka.process$shell.invokeStatic(process.cljc:660)
	at babashka.impl.deps$add_deps$fn__27859.invoke(deps.clj:107)
	at borkdude.deps$_main.invokeStatic(deps.clj:1052)
	at borkdude.deps$_main.doInvoke(deps.clj:882)
	at clojure.lang.RestFn.applyTo(RestFn.java:137)
	at clojure.core$apply.invokeStatic(core.clj:667)
	at babashka.impl.deps$add_deps$fn__27866$fn__27867.invoke(deps.clj:113)
	at clojure.lang.AFn.applyToHelper(AFn.java:152)
	at clojure.lang.AFn.applyTo(AFn.java:144)
	at clojure.core$apply.invokeStatic(core.clj:667)
	at clojure.core$with_bindings_STAR_.invokeStatic(core.clj:1990)
	at clojure.core$with_bindings_STAR_.doInvoke(core.clj:1990)
	at clojure.lang.RestFn.invoke(RestFn.java:425)
	at babashka.impl.deps$add_deps$fn__27866.invoke(deps.clj:113)
	at babashka.impl.deps$add_deps.invokeStatic(deps.clj:113)
	at babashka.main$exec$fn__32736.invoke(main.clj:872)
	at clojure.lang.AFn.applyToHelper(AFn.java:152)
	at clojure.lang.AFn.applyTo(AFn.java:144)
	at clojure.core$apply.invokeStatic(core.clj:667)
	at clojure.core$with_bindings_STAR_.invokeStatic(core.clj:1990)
	at clojure.core$with_bindings_STAR_.doInvoke(core.clj:1990)
	at clojure.lang.RestFn.invoke(RestFn.java:425)
	at babashka.main$exec.invokeStatic(main.clj:824)
	at babashka.main$main.invokeStatic(main.clj:1201)
	at babashka.main$main.doInvoke(main.clj:1148)
	at clojure.lang.RestFn.applyTo(RestFn.java:137)
	at clojure.core$apply.invokeStatic(core.clj:667)
	at babashka.main$_main.invokeStatic(main.clj:1234)
	at babashka.main$_main.doInvoke(main.clj:1226)
	at clojure.lang.RestFn.applyTo(RestFn.java:137)
	at babashka.main.main(Unknown Source)
	at java.base@22/java.lang.invoke.LambdaForm$DMH/sa346b79c.invokeStaticInit(LambdaForm$DMH)
```

### Workaround: add an empty `deps.edn`

``` shell
$ bbin-dev install .

Starting install...

bin    location
─────  ───────────────────────────────────────────────────────
fail1  file:///Users/teodorlu/dev/teodorlu/bbin-testdata/fail1

Install complete.

$ echo '{}' > deps.edn
$ fail1
{:script "fail1", :args nil}
```


## Diagnosis

`fail1` is installed as a babashka wrapper in `~/.local/bin`.

``` shell
$ which fail1
$ cat ~/.local/bin/fail1 
#!/usr/bin/env bb

; :bbin/start
;
; {:coords
;  {:bbin/url "file:///Users/teodorlu/dev/teodorlu/bbin-testdata/fail1"}}
;
; :bbin/end

(require '[babashka.process :as process]
         '[babashka.fs :as fs]
         '[clojure.string :as str])

(def script-root "/Users/teodorlu/dev/teodorlu/bbin-testdata/fail1")
(def script-main-opts-first "-m")
(def script-main-opts-second "fail1/-main")

(def tmp-edn
  (doto (fs/file (fs/temp-dir) (str (gensym "bbin")))
    (spit (str "{:deps {local/deps {:local/root " (pr-str script-root) "}}}"))
    (fs/delete-on-exit)))

(def base-command
  ["bb" "--deps-root" script-root "--config" (str tmp-edn)
        script-main-opts-first script-main-opts-second
        "--"])

(process/exec (into base-command *command-line-args*))
```

`bb` is run with `--deps-root`, which requires a folder with a `deps.edn` file.
In this case, there is no `deps.edn` file.

We can modify the script to figure out what `base-command` and `tmp-edn` is before the script is executed:

``` clojure
;; [... script as before]

(println "tmp-edn:")
(println (slurp tmp-edn))

(println)
(println "base-command:")
(prn base-command)

(process/exec (into base-command *command-line-args*))
```

This gives us some output before the script crashes:

``` shell
tmp-edn:
{:deps {local/deps {:local/root "/Users/teodorlu/dev/teodorlu/bbin-testdata/fail1"}}}

base-command:
["bb" "--deps-root" "/Users/teodorlu/dev/teodorlu/bbin-testdata/fail1" "--config" "/var/folders/_s/yj6rk2hj0llgly4hrb906z4c0000gn/T/bbin158" "-m" "fail1/-main" "--"]
Error building classpath. Manifest file not found for local/deps in coordinate #:local{:root "/Users/teodorlu/dev/teodorlu/bbin-testdata/fail1"}
[... crash as above ...]
```

## Alternatives

### Can bb be run without requiring a `deps.edn` file?

Here's `bb --help`:

``` shell
$ bb --help
Babashka v1.3.190

Usage: bb [svm-opts] [global-opts] [eval opts] [cmdline args]
or:    bb [svm-opts] [global-opts] file [cmdline args]
or:    bb [svm-opts] [global-opts] task [cmdline args]
or:    bb [svm-opts] [global-opts] subcommand [subcommand opts] [cmdline args]

Substrate VM opts:

  -Xmx<size>[g|G|m|M|k|K]  Set a maximum heap size (e.g. -Xmx256M to limit the heap to 256MB).
  -XX:PrintFlags=          Print all Substrate VM options.

Global opts:

  -cp, --classpath  Classpath to use. Overrides bb.edn classpath.
  --debug           Print debug information and internal stacktrace in case of exception.
  --init <file>     Load file after any preloads and prior to evaluation/subcommands.
  --config <file>   Replace bb.edn with file. Defaults to bb.edn adjacent to invoked file or bb.edn in current dir. Relative paths are resolved relative to bb.edn.
  --deps-root <dir> Treat dir as root of relative paths in config.
  --prn             Print result via clojure.core/prn
  -Sforce           Force recalculation of the classpath (don't use the cache)
  -Sdeps            Deps data to use as the last deps file to be merged
  -f, --file <path> Run file
  --jar <path>      Run uberjar

Help:

  help, -h or -?     Print this help text.
  version            Print the current version of babashka.
  describe           Print an EDN map with information about this version of babashka.
  doc <var|ns>       Print docstring of var or namespace. Requires namespace if necessary.

Evaluation:

  -e, --eval <expr>    Evaluate an expression.
  -m, --main <ns|var>  Call the -main function from a namespace or call a fully qualified var.
  -x, --exec <var>     Call the fully qualified var. Args are parsed by babashka CLI.

REPL:

  repl                 Start REPL. Use rlwrap for history.
  socket-repl  [addr]  Start a socket REPL. Address defaults to localhost:1666.
  nrepl-server [addr]  Start nREPL server. Address defaults to localhost:1667.

Tasks:

  tasks       Print list of available tasks.
  run <task>  Run task. See run --help for more details.

Clojure:

  clojure [args...]  Invokes clojure. Takes same args as the official clojure CLI.

Packaging:

  uberscript <file> [eval-opt]  Collect all required namespaces from the classpath into a single file. Accepts additional eval opts, like `-m`.
  uberjar    <jar>  [eval-opt]  Similar to uberscript but creates jar file.
  prepare                       Download deps & pods defined in bb.edn and cache their metadata. Only an optimization, this will happen on demand when needed.

In- and output flags (only to be used with -e one-liners):

  -i                 Bind *input* to a lazy seq of lines from stdin.
  -I                 Bind *input* to a lazy seq of EDN values from stdin.
  -o                 Write lines to stdout.
  -O                 Write EDN values to stdout.
  --stream           Stream over lines or EDN values from stdin. Combined with -i or -I *input* becomes a single value per iteration.

Tooling:

  print-deps [--format <deps | classpath>]: prints a deps.edn map or classpath
    with built-in deps and deps from bb.edn.

File names take precedence over subcommand names.
Remaining arguments are bound to *command-line-args*.
Use -- to separate script command line args from bb command line args.
When no eval opts or subcommand is provided, the implicit subcommand is repl.
```
