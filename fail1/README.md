# `fail1`: a minimal bbin-compatible script

This folder reproduces a known limitation in `babashka/bbin` per 2024-06-19:

https://github.com/babashka/bbin/issues/78

## Expected: this script can be installed.

## Actual: crash when we try to install.

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

## Workaround: add an empty `deps.edn`

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
