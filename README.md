
The goal of this project is to make an F# wrapper
for Gio UI library which is written in Go.

# Implementation (plan)

To make Gio functions accessible to .NET we will use cgo.
We will create a Go project and wrap every Gio function
which we want to call from .NET in a new exported function. 
Since exported functions can't return Go structs
we have to wrap each struct in `cgo.Handle` and return
the handle as `uintptr` (because `cgo.Handle` can't be exported).

The handles are produced by `cgo.NewHandle` function 
and destroyed by `cgo.DeleteHandle` method.
To ensure that handle is properly destroyed when not used in .NET
we will wrap it in a subclass of `SafeHandleZeroOrMinusOneIsInvalid`
which releases it with `cgo.DeleteHandle`.
According to `cgo.NewHandle` source code `0` is not valid for `cgo.Handle`
but `-1` seems fine. So `SafeHandleZeroOrMinusOneIsInvalid` is not perfect
fit, but it will hopefully be fine.
On the other side what can be a problem is that on platforms where `uintptr`
has only 32 bits only `2^32-1` handles can be created.
If an application runs at 120 fps and creates 100 handles per frame
then it will run out of handles and crash after 99 hours.

When Gio returns `event.Event` to an application
the application must type switch on it to find out a concrete event type.
To support this in .NET we write and export a Go function which type switches
and returns an index of a case with the matching type
or `-1` if there was no match. With this function we can determine
more precise types of values associated with handles.

Gio package `layout` contains several functions with `func` parameter.
In .NET we would like to call them with lambda functions.
To pass lambda functions to Go we can convert them to delegates
and pass function pointers for these delegates.
It's important to keep these delegates from being collected by GC
while they are used in Go. The problem is that we don't
know how long they will be used in Go.
In theory, we could use Go finalizers to determine
that a delegate is not needed anymore but according to
[The absurd cost of finalizers in Go](https://lemire.me/blog/2023/05/19/the-absurd-cost-of-finalizers-in-go/)
it seems that the function `runtime.SetFinalizer` is really slow
and calling it several times per each frame would impact the performance.

So our current plan is to completely rewrite packages `layout` and `widget` to F#.
Then there will be no need for passing .NET lambdas to Go.

TODO: What about panics? What happens if we call Go function from .NET
and it panics?
