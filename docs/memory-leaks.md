# Hunting Memory Leaks

Memory leaks can be a big issue with gevent workers because several tasks share the same python process.

Thankfully, MRQ provides tools to track down such issues. Memory usage of each worker is graphed in the dashboard and makes it easy to see if memory leaks are happening.

When a worker has a steadily growing memory usage, here are the steps to find the leak:

 * Check which jobs are running on this worker and try to isolate which of them is leaking and on which queue
 * Start a dedicated worker with ```--trace_memory --greenlets 1``` on the same queue : This will start a worker doing one job at a time with memory profiling enabled. After each job you should see a report of leaked object types.
 * Find the most unique type in the list (usually not 'list' or 'dict') and restart the worker with ```--trace_memory --greenlets 1 --trace_memory_type=XXX --trace_memory_output_dir=memdbg``` (after creating the directory memdbg).
 * There you will find a graph for each task generated by [objgraph](https://mg.pov.lt/objgraph/) which is incredibly helpful to track down the leak.

# Using guppy

If you want to get an interactive debugging session to deal with high memory usage, you can use [guppy](http://guppy-pe.sourceforge.net/). Here is how:

First, initialize a REPL with MRQ configured and guppy loaded:

```
$ pip install guppy
$ python
>>> from mrq import config
>>> from mrq.context import set_current_config, run_task
>>> set_current_config(config.get_config(sources=("file", "env"), config_type="run"))
>>> from guppy import hpy
>>> hp = hpy()
```

Then, wrap your memory-intensive task or code around guppy calls

```
>>> hp.setrelheap()  # Used as reference point for memory usage
>>> run_task("tasks.your.MemoryHungryTask", {"a": 1, "b": 2})
>>> h = hp.heap()
```

At this point `h` should contain all the infos you need. You can view an extended debugging session [here](http://smira.ru/wp-content/uploads/2011/08/heapy.html).

```
>>> h
Partition of a set of 300643 objects. Total size = 41626536 bytes.
 Index  Count   %     Size   % Cumulative  % Kind (class / dict of class)
     0 130043  43 15682088  38  15682088  38 str
     1  76123  25  6978416  17  22660504  54 tuple
     2   1015   0  2794024   7  25454528  61 dict of module
     3  20181   7  2583168   6  28037696  67 types.CodeType
     4  20610   7  2473200   6  30510896  73 function
     5   2321   1  2095216   5  32606112  78 type
     6   2319   1  2045160   5  34651272  83 dict of type
     7   1277   0  1162808   3  35814080  86 dict (no owner)
     8   2890   1   918352   2  36732432  88 unicode
     9    494   0   440912   1  37173344  89 dict of class
```

So our task added 41.6M of RAM to the current process. Let's see where it comes from, starting by these 15M of strings:

```
>>> h[0].byvia
Partition of a set of 130043 objects. Total size = 15682088 bytes.
 Index  Count   %     Size   % Cumulative  % Referred Via:
     0   8208   6  3890208  25   3890208  25 '.func_doc', '[0]'
     1  20065  15  3239664  21   7129872  45 '.co_code'
     2  16673  13  1706224  11   8836096  56 '.co_filename'
     3   2398   2  1606864  10  10442960  67 "['__doc__']"
     4  19810  15  1109640   7  11552600  74 '.co_lnotab'
     5    419   0   308392   2  11860992  76 '.func_doc'
     6   4311   3   285232   2  12146224  77 '[1]'
     7   2788   2   167616   1  12313840  79 '[2]'
     8   2153   2   129136   1  12442976  79 '[3]'
     9   1006   1   109560   1  12552536  80 "['__file__']"
<21212 more rows. Type e.g. '_.more' to view.>
```

Here, it seems that surprisingly, most of the strings are actually docstrings. This can happen if you work with large Python modules like scipy or boto. One might consider stripping them manually or with Python's optimized mode.