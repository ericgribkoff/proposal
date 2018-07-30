Title
----
* Author(s): Eric Gribkoff
* Approver: a11r (?)
* Status: Draft
* Implemented in: Core, Python
* Last updated: 07/30/2018
* Discussion at: <google group thread> (filled after thread exists)

## Abstract

gRPC Core library is compatible with fork() in both parent and child processes
when:

1. gRPC's optional fork support is enabled
1. Post-fork in the child process, gRPC Core is completely shutdown via
destroying any and all created gRPC resources followed by a valid invocation of
`grpc_shutdown()`.

To facilitate this for Python users, a gRPC Python post-fork handler will run in
the child process and automatically `close()` all gRPC core resources on behalf
of the user application.


## Background


Python applications, like many of the wrapped gRPC languages, frequently rely on
the `fork()` syscall to achieve concurrency, due to the CPython global
interpreter lock that limits Python's ability to scale CPU-bound workloads via
threading.

gRPC has offered a very limited support for fork(): currently, gRPC Core has
optional fork handlers registered via `pthread_atfork()` that will pause gRPC
Core's internal threads and resume them after the fork. Without these handlers,
gRPC's internal threads may be holding locks when fork() occurs and the child
process will be in an inconsistent state.

However, any invocation of `grpc_init()` may register global state that can lead
to interference between parent and child process after `fork()` (such as the
global epoll fd registered by the epoll1 polling strategy). Further, gRPC Python
creates its own internal threads to handle things like generating user-request
messages, polling gRPC Core completion queues, and monitoring for connectivity
changes. As a result, gRPC Core's existing fork handlers are currently
insufficient to guarantee that gRPC will function post-fork (in either the child
*or* the parent process).


### Existing fork configuration options

gRPC Core defines an environment variable and a compiler macro to configure its
fork support.

The compiler macro is `GRPC_POSIX_FORK_ALLOW_PTHREAD_ATFORK`. Currently only the
gRPC Python build sets this macros.

The environment variable is `GRPC_ENABLE_FORK_SUPPORT`. If the
`GRPC_POSIX_FORK_ALLOW_PTHREAD_ATFORK` macro is defined, the value of
`GRPC_ENABLE_FORK_SUPPORT` defaults to true. Otherwise, its default is false.

gRPC Core's existing fork handlers are only enabled if both
`GRPC_ENABLE_FORK_SUPPORT` is set to true and
`GRPC_POSIX_FORK_ALLOW_PTHREAD_ATFORK` is defined at compile time.



### Related Proposals: 
N/A

## Proposal


A process may fork and use gRPC in the child process if and only if one of the
following conditions is met:

* gRPC has not been loaded (`grpc_init()` has not been invoked) prior to
fork().

OR

* If `grpc_init()` was invoked prior to fork(), the child process *must* destroy
all gRPC resources inherited from the parent process and invoke
`grpc_shutdown()`. Subsequent to this, the child will be able to re-initialize
and use gRPC.

To facilitate gRPC Python applications meeting the above constraints, gRPC
Python will:

1. Register Cython fork handlers to pause and cleanup gRPC Python's own threads
when fork() is invoked.
1. Track all created gRPC Core resources and, in the
child's post-fork handler, explicitly close these resources and invoke
`grpc_shutdown()` to reset the core library's state.
1. In the child process, gRPC Python will not invoke user callbacks on
inherited, in-progress RPCs that become cancelled due to `fork()`. Since the
user application still retains a reference to the (Python) gRPC call object,
user's may directly request the result of the call: gRPC Python will return an
INTERNAL ("failed due to fork") error for such calls.


From the client application's perspective, any attempt to re-use gRPC channels
in the child process will fail, as those channels have been explicitly closed by
the gRPC Python post-fork handler. But the child process is free to create new
gRPC channels, which will trigger a re-initialization of gRPC Core.


### Change to existing default flag values

This proposal would change the default value of `GRPC_ENABLE_FORK_SUPPORT` to
false, even when the `GRPC_POSIX_FORK_ALLOW_PTHREAD_ATFORK` macro is set. The
current default for gRPC Python is true: but this is not a breaking change,
since the fork handlers registered by setting `GRPC_ENABLE_FORK_SUPPORT` to true
are currently insufficient to support fork once `grpc_init()` has been called.


## Rationale


The proposed approach clearly defines the semantics of fork() support and, by
requiring `grpc_shutdown()` to release all held resources before using gRPC in a
forked process, ensures that no resources will be leaked. 

By handling this complexity within the wrapped language layer, gRPC Python is
able to offer support for fork() to its user applications.

Alternatives considered:

1. Handle `fork()` completely in gRPC Core. This would require Core to
explicitly destroy all created objects (channels, lb policies, etc) in the
child's post-fork handler. This introduces significant overhead and complexity
into Core, as opposed to shutting down resources at the wrapped language layer:
gRPC Python channel's already have a `close()` method which does much of the
heavy work required to shutdown wrapped gRPC core resources. Additionally, gRPC
Python (and other wrapped languages, including Ruby) create their own threads.
These threads would break the internal state of the wrapped language library
without additional fork handlers.
1. Allow the child process to continue using gRPC resources created in the
parent, with a set of internal safeguards (such as blocking TCP reads/writes on
inherited FDs) to ensure the parent and child processes do not conflict. This
has the disadvantage of easily resulting in resource leaks: since gRPC Core is
not completely shutdown after fork(), any resources that are not explicitly
cleaned up will continue to exist in the child. Guaranteeing that all resources
are freed without gRPC shutdown is difficult and prone to regression any time
additional global static state is added to the gRPC library.


## Implementation


### gRPC Core


The following additions will be made to gRPC Core's existing fork support. Note
that the runtime changes only apply when gRPC's fork support is enabled (via
`GRPC_ENABLE_FORK_SUPPORT` environment variable) and the fork handlers are only
registered when the `GRPC_POSIX_FORK_ALLOW_PTHREAD_ATFORK` macro is also
defined. Currently only gRPC Python distributions define this macro.

1. Add a new member, `grpc_core::Fork::is_child_of_fork_`, initialized to false.
1. In the child process's post-fork handler, `is_child_of_fork_` will be set to
true.
1. When `is_child_of_fork_` is true, tcp_posix's `do_read()` and `do_write()`
will fail with an INTERNAL error (with message 'Failed due to fork'). The error
handling will propagate analogously to a socket error, e.g., a read return of 0
bytes.
1. When `is_child_of_fork_` is true, epoll1 will silently skip any calls to
`epoll_wait()` or `epoll_ctl()`, and replaces calls to `shutdown()` with
`close()`.

The above changes ensure that the child process will not interfere with the
parent process's use of gRPC, whether by TCP socket or epoll interest set. These
changes also ensure that, without shutting down the gRPC library, child attempts
to use gRPC will fail.


### gRPC Python

When the `GRPC_ENABLE_FORK_SUPPORT` environment variable is set, gRPC Python
will register Cython fork handlers to:


1. Ensure that Python's own internally-created threads are in a safe state
before fork() proceeds.
1. In the child's post-fork handler, gRPC Python will shut down all existing
gRPC core resources.

This ensures that, in the child process, the gRPC library has been completely
shutdown, with all memory previously allocated for gRPC freed.  The child
process is subsequently free to start using gRPC: this will trigger a fresh
`grpc_init()`.



## Open issues (if applicable)


* gRPC Python threads in user-provided callbacks will not be executing in child 
process
* Fork support will only be compatible with TCP posix and epoll1
