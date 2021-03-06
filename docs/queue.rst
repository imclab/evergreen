
.. module:: evergreen.queue

Synchronization primitives: queues
==================================

The queue module implements multi-producer, multi-consumer queues, useful for
exchanging data between tasks.

This module implements API compatible cooperative versions of the different
queue implementations that can be found in the Python standard library.

The module implements three types of queue, which differ only in the order in
which the entries are retrieved.  In a FIFO queue, the first tasks added are
the first retrieved. In a LIFO queue, the most recently added entry is
the first retrieved (operating like a stack).  With a priority queue,
the entries are kept sorted (using the `heapq` module) and the
lowest valued entry is retrieved first.


.. py:class:: Queue(maxsize)

    Constructor for a FIFO queue.  *maxsize* is an integer that sets the upper
    limit on the number of items that can be placed in the queue.  Insertion will
    block once this size has been reached, until queue items are consumed.  If
    *maxsize* is less than or equal to zero, the queue size is infinite.


.. py:class:: PriorityQueue(maxsize)

    Constructor for a LIFO queue.  *maxsize* is an integer that sets the upper
    limit on the number of items that can be placed in the queue.  Insertion will
    block once this size has been reached, until queue items are consumed.  If
    *maxsize* is less than or equal to zero, the queue size is infinite.


.. py:class:: LifoQueue(maxsize)

    Constructor for a priority queue.  *maxsize* is an integer that sets the upper
    limit on the number of items that can be placed in the queue.  Insertion will
    block once this size has been reached, until queue items are consumed.  If
    *maxsize* is less than or equal to zero, the queue size is infinite.

    The lowest valued entries are retrieved first (the lowest valued entry is the
    one returned by ``sorted(list(entries))[0]``).  A typical pattern for entries
    is a tuple in the form: ``(priority_number, data)``.


Queue Objects
-------------

Queue objects (:class:`Queue`, :class:`LifoQueue`, or :class:`PriorityQueue`)
provide the public methods described below.


.. method:: Queue.qsize()

    Return the approximate size of the queue.  Note, qsize() > 0 doesn't
    guarantee that a subsequent get() will not block, nor will qsize() < maxsize
    guarantee that put() will not block.


.. method:: Queue.empty()

    Return ``True`` if the queue is empty, ``False`` otherwise.  If empty()
    returns ``True`` it doesn't guarantee that a subsequent call to put()
    will not block.  Similarly, if empty() returns ``False`` it doesn't
    guarantee that a subsequent call to get() will not block.


.. method:: Queue.full()

    Return ``True`` if the queue is full, ``False`` otherwise.  If full()
    returns ``True`` it doesn't guarantee that a subsequent call to get()
    will not block.  Similarly, if full() returns ``False`` it doesn't
    guarantee that a subsequent call to put() will not block.


.. method:: Queue.put(item[, block[, timeout]])

    Put *item* into the queue. If optional args *block* is true and *timeout* is
    None (the default), block if necessary until a free slot is available. If
    *timeout* is a positive number, it blocks at most *timeout* seconds and raises
    the :exc:`Full` exception if no free slot was available within that time.
    Otherwise (*block* is false), put an item on the queue if a free slot is
    immediately available, else raise the :exc:`Full` exception (*timeout* is
    ignored in that case).


.. method:: Queue.put_nowait(item)

    Equivalent to ``put(item, False)``.


.. method:: Queue.get([block[, timeout]])

    Remove and return an item from the queue. If optional args *block* is true and
    *timeout* is None (the default), block if necessary until an item is available.
    If *timeout* is a positive number, it blocks at most *timeout* seconds and
    raises the :exc:`Empty` exception if no item was available within that time.
    Otherwise (*block* is false), return an item if one is immediately available,
    else raise the :exc:`Empty` exception (*timeout* is ignored in that case).


.. method:: Queue.get_nowait()

    Equivalent to ``get(False)``.

Two methods are offered to support tracking whether enqueued tasks have been
fully processed by daemon consumer tasks.


.. method:: Queue.task_done()

    Indicate that a formerly enqueued task is complete.  Used by queue consumer
    tasks.  For each :meth:`get` used to fetch a task, a subsequent call to
    :meth:`task_done` tells the queue that the processing on the task is complete.

    If a :meth:`join` is currently blocking, it will resume when all items have been
    processed (meaning that a :meth:`task_done` call was received for every item
    that had been :meth:`put` into the queue).

    Raises a :exc:`ValueError` if called more times than there were items placed in
    the queue.


.. method:: Queue.join()

    Blocks until all items in the queue have been gotten and processed.

    The count of unfinished tasks goes up whenever an item is added to the queue.
    The count goes down whenever a consumer task calls :meth:`task_done` to
    indicate that the item was retrieved and all work on it is complete. When the
    count of unfinished tasks drops to zero, :meth:`join` unblocks.

Example of how to wait for enqueued tasks to be completed

::

    def worker():
        while True:
            item = q.get()
            do_work(item)
            q.task_done()

    q = Queue()
    for i in range(num_worker_tasks):
        t = Task(target=worker)
        t.start()

    for item in source():
        q.put(item)
    q.join()       # block until all tasks are done


Exceptions
----------

.. py:exception:: Empty

    Exception raised when non-blocking :meth:`get` (or :meth:`get_nowait`) is called
    on a :class:`Queue` object which is empty.


.. py:exception:: Full

    Exception raised when non-blocking :meth:`put` (or :meth:`put_nowait`) is called
    on a :class:`Queue` object which is full.

