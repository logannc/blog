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