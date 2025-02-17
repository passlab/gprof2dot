# About _gprof2rdf_

This is a Python script that modifies the original gprof2dot by José Fonseca and generates an RDF knowledge graph using the HPC ontology from profiling input.

It can:

  * read output from:
    * [Linux perf](http://perf.wiki.kernel.org/)
    * [Valgrind's callgrind tool](http://valgrind.org/docs/manual/cl-manual.html)
    * [oprofile](http://oprofile.sourceforge.net/)
    * [sysprof](http://www.daimi.au.dk/~sandmann/sysprof/)
    * [xperf](https://msdn.microsoft.com/en-us/windows/hardware/commercialize/test/wpt/index)
    * [VTune Amplifier XE](http://software.intel.com/en-us/intel-vtune-amplifier-xe)
    * [Very Sleepy](http://www.codersnotes.com/sleepy/)
    * [python profilers](http://docs.python.org/2/library/profile.html#profile-stats)
    * [Java's HPROF](http://docs.oracle.com/javase/7/docs/technotes/samples/hprof.html)
    * prof, [gprof](https://sourceware.org/binutils/docs/gprof/)
    * [DTrace](https://en.wikipedia.org/wiki/DTrace)
  * prune nodes and edges below a certain threshold;
  * use an heuristic to propagate time inside mutually recursive functions;
  * use color efficiently to draw attention to hot-spots;
  * work on any platform where Python and Graphviz is available, i.e, virtually anywhere.

# Requirements

  * [Python](http://www.python.org/download/): known to work with version 2.7 and 3.3; it will most likely _not_ work with earlier releases.
  * [rdflib](https://github.com/RDFLib/rdflib): tested with version 6.1.1

## Windows users

  * Download and install [Python for Windows](http://www.python.org/download/)

## Linux users

On Debian/Ubuntu run:

    apt-get install python3 
    pip3 install rdflib

On RedHat/Fedora run

    yum install python3
    pip3 install rdflib


# Download

  * [PyPI](https://pypi.python.org/pypi/gprof2dot/)

        pip install gprof2dot

  * [Standalone script](https://raw.githubusercontent.com/jrfonseca/gprof2dot/master/gprof2dot.py)

  * [Git repository](https://github.com/jrfonseca/gprof2dot)


# Documentation

## Usage

    Usage:
            gprof2dot.py [options] [file] ...
    
    Options:
      -h, --help            show this help message and exit
      -o FILE, --output=FILE
                            output filename [stdout]
      -n PERCENTAGE, --node-thres=PERCENTAGE
                            eliminate nodes below this threshold [default: 0.5]
      -e PERCENTAGE, --edge-thres=PERCENTAGE
                            eliminate edges below this threshold [default: 0.1]
      -f FORMAT, --format=FORMAT
                            profile format: axe, callgrind, hprof, json, oprofile,
                            perf, prof, pstats, sleepy, sysprof or xperf [default:
                            prof]
      --total=TOTALMETHOD   preferred method of calculating total time: callratios
                            or callstacks (currently affects only perf format)
                            [default: callratios]
      -s, --strip           strip function parameters, template parameters, and
                            const modifiers from demangled C++ function names
      -w, --wrap            wrap function names
      --show-samples        show function samples
      -z ROOT, --root=ROOT  prune call graph to show only descendants of specified
                            root function
      -l LEAF, --leaf=LEAF  prune call graph to show only ancestors of specified
                            leaf function
	  --list-functions=SELECT list available functions as a help/preparation  for using the 
	                        -l and -z flags. When selected the program only produces this
							list. SELECT is used with the same matching syntax
							as with -z(--root) and -l(--leaf). Special cases SELECT="+"
							gets the full list, selector starting with "%" cause dump 
							of all available information. 

## Examples

### Linux perf

    perf record -g -- /path/to/your/executable
    perf script | c++filt | gprof2dot.py -f perf 

### oprofile

    opcontrol --callgraph=16
    opcontrol --start
    /path/to/your/executable arg1 arg2
    opcontrol --stop
    opcontrol --dump
    opreport -cgf | gprof2dot.py -f oprofile 

### xperf

If you're not familiar with xperf then read [this excellent article](http://blogs.msdn.com/b/pigscanfly/archive/2009/08/06/stack-walking-in-xperf.aspx) first. Then do:

  * Start xperf as

        xperf -on Latency -stackwalk profile

  * Run your application.

  * Save the data.
`
        xperf -d output.etl

  * Start the visualizer:

        xperf output.etl

  * In _Trace_ menu, select _Load Symbols_. _Configure Symbol Paths_ if necessary.

  * Select an area of interest on the _CPU sampling graph_, right-click, and select _Summary Table_.

  * In the _Columns_ menu, make sure the _Stack_ column is enabled and visible.

  * Right click on a row, choose _Export Full Table_, and save to _output.csv_.

  * Then invoke gprof2dot as

        gprof2dot.py -f xperf output.csv

### VTune Amplifier XE

  * Collect profile data as (also can be done from GUI):

        amplxe-cl -collect hotspots -result-dir output -- your-app

  * Visualize profile data as:

        amplxe-cl -report gprof-cc -result-dir output -format text -report-output output.txt
        gprof2dot.py -f axe output.txt | dot -Tpng -o output.png

See also [Kirill Rogozhin's blog post](http://software.intel.com/en-us/blogs/2013/04/05/making-visualized-call-graph-from-intel-vtune-amplifier-xe-results).

### gprof

    /path/to/your/executable arg1 arg2
    gprof path/to/your/executable | gprof2dot.py 

### python profile

    python -m profile -o output.pstats path/to/your/script arg1 arg2
    gprof2dot.py -f pstats output.pstats 

### python cProfile (formerly known as lsprof)

    python -m cProfile -o output.pstats path/to/your/script arg1 arg2
    gprof2dot.py -f pstats output.pstats 

### Java HPROF

    java -agentlib:hprof=cpu=samples ...
    gprof2dot.py -f hprof java.hprof.txt 

See [Russell Power's blog post](http://rjp.io/2012/07/03/java-profiling/) for details.

### DTrace

    dtrace -x ustackframes=100 -n 'profile-97 /pid == 12345/ { @[ustack()] = count(); } tick-60s { exit(0); }' -o out.user_stacks
    gprof2dot.py -f dtrace out.user_stacks 

    # Notice: sometimes, the dtrace outputs format may be latin-1, and gprof2dot will fail to parse it.
    # To solve this problem, you should use iconv to convert to UTF-8 explicitly.
    # TODO: add an encoding flag to tell gprof2dot how to decode the profile file.
    iconv -f ISO-8859-1 -t UTF-8 out.user_stacks | gprof2dot.py -f dtrace

## Output

The output of gprof2rdf is an RDF knowledge graph that maps functions and callpaths to values obtained from the given profile information, which can be used for querying or merging to create a larger callgraph. The ontology an extension of the normal HPC specificiation and is as follows:

?f `hpc:selfTime`: `xsd:float` - Reflects the amount of the time the function was in itself (not in other function calls)
?f `hpc:totalTime`: `xsd:float` - Reflects the total amount of time the function was being executed
?f `hpc:called` `xsd:integer` - Counts the number of time a function was called
?f `hpc:caller` - Links a caller to a callpath
?f `hpc:upstreamCallPath` - Links a callee to a callpath

?c `hpc:srcFunc` - Indicates the source function (caller) of the given callpath
?c `hpc:destFunc` - Indicates the destination function (callee) of the given callpath
?c `hpc:selfTime`: `xsd:float` - Reflects the amount of the time the callpath was in itself (not in other function calls)
?c `hpc:totalTime`: `xsd:float` - Reflects the total amount of time the callpath was being executed
?c `hpc:called` `xsd:integer` - Counts the number of time a callpath was called


## Listing functions

The flag `--list-functions` permits listing the function entries found in the `gprof` input.
This is intended as a tool to prepare for utilisations with the `--leaf` (`-l`) 
or `--root` (`-z`) flags.

  ~~~
  prof2dot.py -f pstats /tmp/myLog.profile  --list-functions "test_segments:*:*" 
    
  test_segments:5:<module>,
  test_segments:206:TestSegments,
  test_segments:46:<lambda>
  ~~~

  - The selector argument is used with Unix/Bash globbing/pattern matching, in the same
    fashion as performed by the `-l` and `-z` flags.
	  
  - Entries are formatted '\<pkg\>:\<linenum\>:\<function\>'. 
	
  - When selector argument starts with '%', a dump of all available information is 
	performed for selected entries,   after removal of selector's leading '%'. If 
	selector is "+" or "*", the full list of functions is printed.


## Frequently Asked Questions

### How can I generate a complete call graph?

By default `gprof2dot.py` generates a _partial_ call graph, excluding nodes and edges with little or no impact in the total computation time. If you want the full call graph then set a zero threshold for nodes and edges via the `-n` / `--node-thres`  and `-e` / `--edge-thres` options, as:

    gprof2dot.py -n0 -e0


### Why don't the percentages add up?

You likely have an execution time too short, causing the round-off errors to be large.

See question above for ways to increase execution time.

### Which options should I pass to gcc when compiling for profiling?

Options which are _essential_ to produce suitable results are:

  * **`-g`** : produce debugging information
  * **`-fno-omit-frame-pointer`** : use the frame pointer (frame pointer usage is disabled by default in some architectures like x86\_64 and for some optimization levels; it is impossible to walk the call stack without it)

_If_ you're using gprof you will also need `-pg` option, but nowadays you can get much better results with other profiling tools, most of which require no special code instrumentation when compiling.

You want the code you are profiling to be as close as possible as the code that you will
be releasing. So you _should_ include all options that you use in your release code, typically:

  * **`-O2`** : optimizations that do not involve a space-speed tradeoff
  * **`-DNDEBUG`** : disable debugging code in the standard library (such as the assert macro)

However many of the optimizations performed by gcc interfere with the accuracy/granularity of the profiling results.  You _should_ pass these options to disable those particular optimizations:

  * **`-fno-inline-functions`** : do not inline functions into their parents (otherwise the time spent on these functions will be attributed to the caller)
  * **`-fno-inline-functions-called-once`** : similar to above
  * **`-fno-optimize-sibling-calls`** : do not optimize sibling and tail recursive calls (otherwise tail calls may be attributed to the parent function)

If the granularity is still too low, you _may_ pass these options to achieve finer granularity:

  * **`-fno-default-inline`** : do not make member functions inline by default merely because they are defined inside the class scope
  * **`-fno-inline`** : do not pay attention to the inline keyword
Note however that with these last options the timings of functions called many times will be distorted due to the function call overhead. This is particularly true for typical C++ code which _expects_ that these optimizations to be done for decent performance.

See the [full list of gcc optimization options](http://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html) for more information.

# Links

See the [wiki](https://github.com/jrfonseca/gprof2dot/wiki) for external resources, including complementary/alternative tools.
