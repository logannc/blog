+++
title = "Verifying Atomic Python Writes"
description = "Multithreading is hard. How do we know that writes from different python threads to the same file object are atomic?"
date = 2018-10-28
in_search_index = true
render = true
[extra]
alttext = "Python, Multithreading, and Atomicity!"
[taxonomies]
tags = ['python', 'atomic', 'posix', 'threads']
categories = ['programming']
+++

### Background

Anyone who has used Python long enough, at a high enough level will eventually need to start using multiple threads. In general, Python has a somewhat complicated story around threads due to the GIL. It's easy enough to use threads, but using them *well* requires knowing a bit more about the internal implementation of Python. 

One exception to this is writing to a single file from multiple threads. If multiple threads all have access to the same file object, as long as each thread's write is expressed as a single `file_object.write(...)` call, everything works as expected.


Links:

https://github.com/python/cpython/blob/fc512e3e0663f7f325862fcd42aef765fd34a453/Modules/_io/fileio.c#L840
https://github.com/python/cpython/blob/74a8b6ea7e0a8508b13a1c75ec9b91febd8b5557/Python/fileutils.c#L1512
http://pubs.opengroup.org/onlinepubs/9699919799/functions/V2_chap02.html#tag_15_09_07
http://man7.org/linux/man-pages/man2/write.2.html
https://lkml.org/lkml/2014/3/3/533

https://stackoverflow.com/questions/2220525/is-fwrite-atomic
https://stackoverflow.com/questions/11414191/what-are-the-main-differences-between-fwrite-and-write
