# Threads

## Stack layout

Assume the program call func(arg0, arg1, ..., argN); When entering the func, the stack layout should be

<pre>
    +-------------+
    | argN        |
c   +-------------+
a   | ...         |
l   +-------------+
l   | arg1        |
e   +-------------+
r   | arg0        |
    +-------------+
    | return addr |
    +-------------+
    | old ebp     |
-.-.+-------------+ <-- ebp, esp
c
a
l
l
e
e

</pre>

## Second pointer
``int join(void **stack)`` passes a second pointer to the system call ``join``. The user stack after the call is
<pre>
    +---------------------+
    |                     |
    |                     |
    +---------------------+<--+ stack
    | ...                 |   |
    +---------------------+   |
    | void* stack         |---+
    +---------------------+<--+
    |  ...                |   |
    +---------------------+   |
    | &stack              |---+
    +---------------------+
    | return addr         |
    +---------------------+
    | old $ebp            |
    +---------------------+ <-- esp, ebp
</pre>

In ``sys_join`` [kernel/sysproc.c](../kernel/sysproc.c), 

    void **stack;
    if (argptr(0, (char**)&stack, sizeof(void**)) < 0) {
      ...
    return join(stack);

Now the two ``stack`` in both the user space and the system call are identical. There is a trick that we convert the type of the address of ``stack`` to ``(char**)`` so that we can leverage the ``argptr`` which is used to fetch a normal pointer.


# Important changes in clone, wait, and join

The ``clone`` function is similar to ``fork``, but we need

1. Directly assign ``proc->pgdir`` to ``np->pgdir`` (threads share the page table) instead of ``copyuvm`` (creating a new page table).
1. Set up the new user stack and change the ``$esp``.
1. Change the ``$eip`` to the address of the ``start_routine``.

In ``wait``, when ``p->pgdir == proc->pgdir``, we do not kill the process. The work is left for ``join``. In ``join``, when ``p->pgdir != proc->pgdir``, we do not kill the process. The work is left for ``wait``.

When ``clone``, ``wait`` and ``join``, we need to track the count of references to ``pgdir``. The count table is defined in [kernel/proc.c](../kernel/proc.c).

    struct pgdir_refs {
      void* addr;
      int count;
    };

    struct {
      struct spinlock lock;
      struct pgdir_refs refs[NPROC];
    } pgdir_table;

# Relationship with fork and exec
## Call fork in a thread
If ``fork()`` is called in a thread (parent), the child have a different page table. If the parent exits before the child, the child will be attached to its grandparent (i.e. the one that cloned the parent). The new relationship between the grandpa and the child is fork.

If ``exex()`` is called in a thread, the thread will be replaced by the new program. The relationship between the thread and its parent changes from clone to fork. Thus ``fork()`` then ``exec()`` in children is identical to ``clone()`` then ``exec()``.



# Stack memory management
When create a thread, we need to allocate one page for the stack. The stack also has to be page aligned. To meeting the requirement, I

1. malloc two pages. 
1. set the stack pointer to the lowest page aligned address.
1. set stack[-1] the address of the memory allocated (``ptr_s``), so that we can retrieve the pointer when we free the stack.

The code snippet is

    uint page_size = 4096U;
    ptr_s = malloc(page_size * 2);
    ...
    stack = (void*)(((uint)ptr_s + page_size) & ~0xFFF);
    ((uint*)stack)[-1] = (uint)ptr_s; 

And the stack layout looks like

<pre>
    +---------------+
    | redundent     |
    +---------------+
    |               |
    |               |
    +---------------+ <-- stack
    | ptr_s         | --+
    +---------------+   |
    | ...           |   |
    +---------------+   |
    | redundent     |   |
    +---------------+ <-- ptr_s
</pre>

On ``thread_join``, we can find ``ptr_s`` from the ``stack`` we get and ``free(ptr_s)``.


# Shared memory in threads
## Semantics
1. All threads in the same process have one count in the shared pages.
1. When clone, copy the ``pgdir, shm, shm_key_mask, *shm_va[MAX_SHM_KEY]``. Do not add reference counts.
1. When join, release the shared memory only when the thread is the last one to exit.
1. The created thread can access its parent's shared memory with the key. The ``shmgetat(key, ANY)`` should return the virtual address of the shared memory immediately without allocating new pages.
