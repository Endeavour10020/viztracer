# VizTracer

[![build](https://github.com/gaogaotiantian/viztracer/workflows/build/badge.svg)](https://github.com/gaogaotiantian/viztracer/actions?query=workflow%3Abuild)  [![pypi](https://img.shields.io/pypi/v/viztracer.svg)](https://pypi.org/project/viztracer/)

VizTracer is a deterministic debugging/profiling tool that can trace and visualize python code. The major data VizTracer displays is FEE(function entry/exit), or equivalently, the call stack. 

Unlike traditional flame graph, which is normally generated by sampling profiler, VizTracer can display every function executed and the corresponding entry/exit time from the beginning of the program to the end, which is helpful for programmers to catch sporatic performance issues. 

With VizTracer, the programmer can intuitively understand what their code is doing and how long each function takes.  

You can take a look at the [demo](http://www.minkoder.com/viztracer/result.html) result of an example program running recursive merge and quick sort algorithm.

[trace viewer](https://chromium.googlesource.com/catapult) is used to display the stand alone html data.

VizTracer also supports json output that complies with Chrome trace event format, which can be loaded using [perfetto](https://ui.perfetto.dev/)

## Requirements

VizTracer requires python 3.5+. No other package is needed. For now, VizTracer binary on pip only supports CPython + Linux. However, in theory the source code can build on Windows/MacOS.

## Install

The prefered way to install VizTracer is via pip

```
pip install viztracer
```

You can also download the source code and build it yourself.

## Usage

There are a couple ways to use VizTracer

### Command Line

The easiest way to use VizTracer it through command line. Assume you have a python script to profile and the normal way to run it is:

```
python3 my_script.py
```

You can simply use VizTracer as 

```
python3 -m viztracer my_script.py
```

which will generate a ```result.html``` file in the directory you run this command. Open it in browser and there's your result.

If your script needs arguments like 

```
python3 my_script.py arg1 arg2
```

Just feed it as it is to VizTracer

```
python3 -m viztracer my_script.py arg1 arg2
```

You can also specify the tracer to be used in command line by passing --tracer argument. c tracer is the default value, you can use python tracer instead

```
python3 -m viztracer --tracer c my_script.py
python3 -m viztracer --tracer python my_script.py
```

You can specify the output file using -o or --output_file argument. The default output file is result.html. Two types of files are supported, html and json.
```
python3 -m viztracer -o other_name.html my_script.py
python3 -m viztracer -o other_name.json my_script.py
```

### Inline

Sometimes the command line may not work as you expected, or you do not want to profile the whole script. You can manually start/stop the profiling in your script as well.

First of all, you need to import ```VizTracer``` class from the package, and make an object of it.

```python
from viztracer import VizTracer

tracer = VizTracer()
```

If your code is executable by ```exec``` function, you can simply call ```tracer.run()```

```python
tracer.run("import random;random.randrange(10)")
```

This will as well generate a ```result.html``` file in your current directory. You can pass other file path to the function if you do not like the name ```result.html```

```python
tracer.run("import random; random.randrange(10)", output_file = "better_name.html")
```

When you need a more delicate profiler, you can manually enable/disable the profile using ```start()``` and ```stop()``` function.

```python
tracer.start()
# Something happens here
tracer.stop()
tracer.save() # also takes output_file as an optional argument
```

With this method, you can only record the part that you are interested in

```python
# Some code that I don't care
tracer.start()
# Some code I do care
tracer.stop()
# Some code that I want to skip
tracer.start()
# Important code again
tracer.stop()
tracer.save()
```

**It is higly recommended that ```start()``` and ```stop()``` function should be in the same frame(same level on call stack). Problem might happen if the condition is not met**

### Display Result

By default, VizTracer will generate a stand alone HTML file which you can simply open with Chrome(maybe Firefox?). The front-end uses trace-viewer to show all the data. 

However, you can generate json file as well, which complies to the chrome trace event format. You can load the json file on [perfetto](https://ui.perfetto.dev/), which will replace the deprecated trace viewer in the future. 

At the moment, perfetto did not support locally stand alone HTML file generation, so I'm not able to switch completely to it. The good news is that once you load the perfetto page, you can use it even when you are offline. 


### Trace Filter

Sometimes your code is really complicated or you need to run you program for a long time, which means the parsing time would be too long and the HTML/JSON file would be too large. There are ways in viztrace to filter out the data you don't need. 

The filter mechanism only works in C tracer, and it works at tracing time, not parsing time. That means, using filters will introduce some extra overhead while your tracing, but will save significant memory, parsing time and disk space. 

Currently we support two kinds of filters:

#### max_stack_depth

```max_stack_depth``` is a straight forward way to filter your data. It limits the stack depth viztracer will trace, which cuts out deep call stacks, including some nasty recursive calls. 

You can specify ```max_stack_depth``` in command line:

```
python3 -m viztracer --max_stack_depth 10 my_script.py
```

Or you can pass it as an argument to the ```VizTracer``` object:

```python
from viztracer import VizTracer

tracer = VizTracer(max_stack_depth=10)
```


#### include_files and exclude_files

There are cases when you are only interested in functions in certain files. You can use ```include_files``` and ```exclude_files``` feature to filter out data you are not insterested in. 

When you are using ```include_files```, only the files and directories you specify are recorded. Similarly, when you are using ```exclude_files```, files and directories you specify will not be recorded. 

**IMPORTANT: ```include_files``` and ```exclude_files``` can't be both spcified. You can only use one of them.**

**If a function is not recorded based on ```include_files``` or ```exclude_files``` rules, none of its descendent functions will be recorded, even if they match the rules**

You can specify ```include_files``` and ```exclude_files``` in command line, but they can take more than one argument, which will make the following command ambiguous:

```
# Ambiguous command which should NOT be used
python3 -m viztracer --include_files ./src my_script.py
```

Instead, when you are using ```--include_files``` or ```--exclude_files```, ```--run``` should be passed for the command that you actually want to execute:

```
# --run is used to solve ambiguity
python3 -m viztracer --include_files ./src --run my_script.py
```

However, if you have some other commands that can separate them and solve ambiguity, that works as well:

```
# This will work too
python3 -m viztracer --include_files ./src --max_stack_depth 5 my_script.py
```

You can also pass a ```list``` as an argument to ```VizTracer```:

```python
from viztracer import VizTracer

tracer = VizTracer(include_files=["./src", "./test/test1.py"])
```

### Choose Tracer

The default tracer for current version is c tracer, which introduce a relatively small overhead(worst case 2-3x) but only works for CPython on Linux. However, if there's other reason that you would prefer a pure-python tracer, you can use python tracer using ```tracer``` argument when you initialize ```VizTracer``` object.

```python
tracer = VizTracer(tracer="python")
```

**python tracer will be deprecated because of the performance issue in the future**

#### Cleanup of c Tracer

The interface for c trace is almost exactly the same as python tracer, except for the fact that c tracer does not support command line run now. However, to achieve lower overhead, some optimization is applied to c tracer so it will withhold the memory it allocates for future use to reduce the time it calls ```malloc()```. If you want the c trace to free all the memory it allocates while collecting trace, use

```python
tracer.cleanup()
```

## Performance

Overhead is a big consideration when people choose profilers. VizTracer now has a similar overhead as native cProfiler. It works slightly worse in the worst case(Pure FEE) and better in easier case because even though it collects some extra information than cProfiler, the structure is lighter. 

Admittedly, VizTracer is only focusing on FEE now, so cProfiler also gets other information that VizTracer does not acquire.

An example run for test_performance with Python 3.8 / Ubuntu 18.04.4 on Github VM

```
fib       (10336, 10336): 0.000852800 vs 0.013735200(16.11)[py] vs 0.001585900(1.86)[c] vs 0.001628400(1.91)[cProfile]
hanoi     (8192, 8192): 0.000621400 vs 0.012924899(20.80)[py] vs 0.001801800(2.90)[c] vs 0.001292900(2.08)[cProfile]
qsort     (10586, 10676): 0.003457500 vs 0.042572898(12.31)[py] vs 0.005594100(1.62)[c] vs 0.007573200(2.19)[cProfile]
slow_fib  (1508, 1508): 0.033606299 vs 0.038840998(1.16)[py] vs 0.033270399(0.99)[c] vs 0.032577599(0.97)[cProfile]
```

## Limitations

VizTracer uses ```sys.setprofile()``` for its profiler capabilities, so it will conflict with other profiling tools which also use this function. Be aware of it when using VizTracer.

## Bugs/Requirements

Please send bug reports and feature requirements through [github issue tracker](https://github.com/gaogaotiantian/viztracer/issues). VizTracer is currently under development now and it's open to any constructive suggestions.

## License

Copyright Tian Gao, 2020.

Distributed under the terms of the  [Apache 2.0 license](https://github.com/gaogaotiantian/viztracer/blob/master/LICENSE).