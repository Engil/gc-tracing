This repository tracks ongoing work about replacing OCaml's instrumented runtime with a tracing system, inspired by OCaml multicore's tracing system.

## How tracing works

The tracing runtime works by dispatching events to a buffer and flushing these periodically.
Within the runtime code, various functions are available to start recording events, attached with a timestamp sourced from a monotonic clock where each tick is a nanosecond that passed.

Once the event buffer is full, events will be serialized and pushed onto a file.

Tracing should be pause-able, stoppable and restartable, as well as streamable.


## Implementations

We currently have two different approaches to tracing implemented.
One is based on a simple port of OCaml multicore's tracing system, and the other one is making use of the CTF format.

Both can be tried easily, by compiling the following branches and running your compiled program with the following env variable:
`OCAML_EVENTLOG_ENABLED=1` and `OCAML_EVENTLOG_FILE=outputfile`.

Refer to the following sections for more information on where to find the implementation and how to read the generated files.

### Catapult/Chrome

OCaml multicore make use of Chrome's catapult, a tracer available in-browser recording and displaying traces informations.
The viewer can load its own format, written in either Json or Linux's ftrace format.
It has been ported to OCaml trunk and made to match what the instrumented runtime used to collect.

[OCaml multicore tracing implementation](https://github.com/ocaml-multicore/ocaml-multicore/blob/master/byterun/eventlog.c)

[Trace Event Format](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/preview)

[Port to OCaml trunk of multicore's tracing](https://github.com/Engil/ocaml/blob/eventlog/runtime/eventlog.c)

Sample trace for the trunk port can be found in the `traces` directory, `numal-lu-decomposition.json`, you can open it through Chrome at `chrome://tracing`.

### Common Trace Format

Common Trace Format is a fast, binary and streamable format.

It is meant as a generalistic trace format, allowing reuse of existing tooling (like [babeltrace](https://diamon.org/babeltrace/) or [Trace Compass](https://www.eclipse.org/tracecompass/). `babeltrace` also enable painless conversion from one trace format to the other, providing a simple CTF reader.

[Common Tracing Format specification](https://diamon.org/ctf/)

[Tracing implemented with CTF output on OCaml trunk](https://github.com/Engil/ocaml/blob/ctf/runtime/eventlog.c)

Note that to read the file you should as well provide the `metadata` file on this repository, describing the CTF schema used by OCaml. (at the moment this file isn't generated by OCaml itself.)

See `scripts/ctf_to_catapult.py` to convert an OCaml CTF trace to Chrome's catapult format.

You can as well read the traces with `babeltrace` (available in most Linux distributions):

```
engil@monsieurlordinateur:~/Dev/gc-tracing$ babeltrace traces/ctf/ | head
[01:00:00.000926193] (+?.?????????) 0 counter: { kind = ( "request_major/adjust_gc_speed" : container = 13 ), count = 0 }
[01:00:00.000926254] (+0.000000061) 0 entry: { phase = ( "major_roots/global" : container = 13 ) }
[01:00:00.000926465] (+0.000000211) 0 entry: { phase = ( "major_roots/finalised" : container = 16 ) }
[01:00:00.000932746] (+0.000006281) 0 exit: { phase = ( "major_roots/finalised" : container = 16 ) }
[01:00:00.000932780] (+0.000000034) 0 entry: { phase = ( "major/sweep" : container = 9 ) }
[01:00:00.001242133] (+0.000309353) 0 exit: { phase = ( "major/sweep" : container = 9 ) }
[01:00:00.001242170] (+0.000000037) 0 entry: { phase = ( "major/mark_roots" : container = 10 ) }
[01:00:00.001246365] (+0.000004195) 0 exit: { phase = ( "major/mark_roots" : container = 10 ) }
[01:00:00.001246409] (+0.000000044) 0 entry: { phase = ( "major/mark_main" : container = 11 ) }
[01:00:00.001246657] (+0.000000248) 0 exit: { phase = ( "major/mark_main" : container = 11 ) }
```

## Comparing CTF and Catapult

Catapult's viewer is pretty useful, but the available formats for serialization are a bit heavy
CTF's binary format make it easier to write fast serializers in a tight package.

Some early numbers from a payload size point of view:


```
engil@monsieurlordinateur:~/Dev/sandmark$ find _build/ -type f -name '*.catapult' -exec du -ch {} + | grep total$
6,0G	total
engil@monsieurlordinateur:~/Dev/sandmark$ find _build/ -type f -name '*.ctf' -exec du -ch {} + | grep total$
1006M	total
```

This CTF schema can be greatly improved upon in terms of size (see the `metadata` file), as well as execution time.
Proper benchmark for overhead (comparing `trunk` vs `ctf` vs `catapult`) are coming sometime soon.

CTF's streaming approach also makes it preferrable to trace OCaml's multicore's runtime vs the "one json to rule them all" approach in the Catapult format. Each domain could easily output their own stream and the metadata file could take care of cross referencing every moving part from within the program's lifetime.