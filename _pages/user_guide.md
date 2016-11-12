---
layout: page
permalink: user-guide/
---
# User Guide

This page is intended to help users leverage all the features of DINAMITE.

## Contents

#### [Instrumentation filtering](#instrumentation-filtering)

#### [Log format](#log-format)

#### [String maps](#string-maps)


<hr>

# <a name="instrumentation-filtering"></a> Instrumentation filtering 

By default DINAMITE will instrument every memory access in your program. However, often you only want to instrument parts of the code, and not all memory accesses, but simply function entry/exit timestamps. 

In order to instrument only parts of your code, and specific events, DINAMITE supports function filtering.
To leverage this, you need to provide a filter file. The filter basically works as a white list for functions that are allowed to be instrumented.
It is stored in JSON format and looks something like this:

```
{
    "minimum_function_size" : 10,
    "function_size_metric" : "LOC_PATH",
    "check_small_function_loops" : true,

    "blacklist" : {
        "function_filters" : [
            "function_that_wont_be_instrumented1",
            "function_that_wont_be_instrumented2"
        ],
        "file_filters" : [
            "wont_be_instrumented.c"
        ]
    },
    "whitelist": {
        "file_filters" : [
        ],
        "function_filters" : {
            "*" : {
                "events" : [
                    "function",
                    "alloc"
                ]

            },
            "function_with_an_interesting_access_pattern" : {
                "events" : [
                    "access",
                    "function"
                ]
            },
            "*lock*" : {
                "events" : [
                    "function"
                ],
                "arguments" : "*"
            },
            "interesting_second_argument" : {
                "events" : [
                    "function"
                ],
                "arguments" : [ 1 ]
            }
        }
    }
}
```

The top level objects in the filter JSON specification are:

- `minimum_function_size` (integer) - Anything below this size (in specified units, see next) will **not** be instrumented.
- `function_size_metric` - Can have three different values: `IR`, `LOC` and `LOC_PATH`. This tells the compiler what metric we would like to use for size-based
    filtering. Respectively, the three metrics are: number of LLVM IR instructions, total number of lines of code in a function and
    number of lines of code in the longest execution path through the function.
- `check_small_function_loops` - if set to true, it will **enable** instrumentation of functions smaller than the specified size which **contain a loop**
- `blacklist` - list of files and functions which will be ignored when instrumenting
- `whitelist` - list of files and functions which will get instrumented, along with more detailed filtering of functions

Any field that calls for naming an entity in the code allows for simple globbing (using `*` as a wildcard for parts of the string)

## Whitelisting and blacklisting

Whitelist and blacklist entries have a priority ordering as follows:

- Everything that is in the blacklist will **automatically** get ignored.
- If the whitelists are empty, everything is instrumented.
- As soon as anything is specified in the whitelist, everything else is ignored
- Functions that are smaller than the specified minimum size will get ignored, unless they match anything other than `"*"` in the whitelist.

## Whitelist function entry format

The `whitelist->function_filters` object is keyed with function names. Its contents are an array of `events` which can contain the strings
`"access"`, `"function"` and `"alloc"`, telling DINAMITE to instrument memory access, function and allocation events in the specified function.
When instrumenting a function one can also specify arguments to be logged. In practice, we found this to be very helpful when logging lock behaviour
in multithreaded programs. Locking routines usually accept lock pointers as arguments, and with argument logging one can get all the pointer values
that get passed.

## Using filters in DINAMITE

In order to tell DINAMITE about the function filters, at the time you compile your program with DINAMITE, you will need to set the environment variable `DIN_FILTERS` to point to the JSON document, like so:

`DIN_FILTERS="/path/to/function_filter.json" make -j 4`

If you don't set `DIN_FILTERS`, DINAMITE assumes that you want to instrument all the events in your program. **It is safest to provide the full path of the `function_filters.json` file.** If you are building a complex project the build might be performed in multiple directories. If only the top-level directory has the filters file, the files in the other directories will not be filtered correctly.

With C++ code, function names are typically mangled. To find a real function name, run compilation once. The filtering tool will output `function_sizes.json` which will contain the names of all encountered functions and their respective sizes (defaults to LOC_PATH metric). You can use these to fill your filter list properly.

# <a name="log-format"></a> Log format

DINAMITE's log format is not enforced. If you don't like the way it is,
writing a new logging library takes very little effort! If you do that,
we'd love to hear how you did it and why!

The default implementation in the binary logging library differentiates
between three types of events: function, allocation and access.
This is, in fact, all the event types that are available from DINAMITE.
When you instrument a program, the compiler adds calls to external logging
functions in the appropriate places and passes them all the relevant information.

(You can follow along by opening `<dinamite_root>/library/binaryinstrumentation.h` )

All three log event types are encased in a union which contains a field to identify the event type.

All three log event types also have a shared field called `thread_id` which, as you've
already guessed, contains the ID of the thread the event happened on. This is the only field
that doesn't come from DINAMITE's compiler (not available at compile time). The implementation
in `binaryinstrumentation.h` can be used for reference if you want to make a new 
thread-aware logging implementation..

The following field descriptions will omit the `thread_id` field, so don't get confused, it's still
there.

## Function events

These events contain the type of event (`fn_event_type`) which can have a value of either `FN_BEGIN` or `FN_END` (defined in the same file) and the ID of the function (see string maps).

## Allocation events

Allocation events are described with the code location of where the allocation
happened and information about the type, size and number of allocated elements.
They also contain the address of the beginning of the allocated memory region.

The field `size` is expressed in bytes.

The `type` and `file` fields are string IDs, which will correspond with the emitted
JSON string maps.

## Access events

Access events each describe a single memory access.

They contain the pointer to the accessed memory location (`ptr`), value - encoded as a union
of all possible primitive types, type of access ('r' for reads, 'w' for writes, and 'a' for
special argument logging events), three fields for code location, type of accessed data and
the name of the variable (both corresponding to the string maps).

Variable names can pertain to single scalar variables, in which case, their name in the maps
will be whatever is visible to LLVM in its IR. Besides that, they can be fields of a complex
data type (classes or structs), in which case, their name will be of the format "ClassName.fieldName".

In our experience, this is the most meaningful way to encode variable name information.

# <a name="string-maps"></a> String maps

In order to keep the size of logs as small as possible, DINAMITE encodes all strings encountered
while instrumenting as integer IDs. All strings fall into one of the 4 categories: type,
variable, function and file names.
When compiling, DINAMITE stores these string to integer mappings in 4 JSON files.

The default location of these files is the current build directory. This, however,
means that build systems in which the compiler is invoked from different locations
will store maps in multiple directories. **This is incorrect behaviour**, because
DINAMITE relies on reading the state of these maps from the previous invocations.
To avoid this problem, when invoking the build, provide the path in which to store
string maps as `$DIN_MAPS` like so:

```
DIN_MAPS=/path/to/maps make
```

After the build, `/path/to/maps` will contain 4 files: `map_sources.json`, `map_functions.json`,
`map_types.json` and `map_variables.json`, each containing string to integer ID mappings
for its respective string category.
