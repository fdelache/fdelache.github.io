---
title: How Ruby Objects are represented in memory?
categories: [ruby, memory, internals]
author: Franck
authorBluesky: https://bsky.app/profile/franck.isalazy.dev
image: /images/snow-fall.webp
gravatar: https://gravatar.com/avatar/3b7898bd27b4e323233d647c63e4d87e?size=150
gravatarAlt: Profile picture of Franck Delache
postFooter: 
---

I recently spent some time exploring the Ruby VM internals to better understand how the virtual machine works under the hood. As a Ruby developer, the sole existence of the VM is very convenient: no need to worry about memory management, it’s all done by the VM. But how does it actually work? In this post, we’ll discover how the virtual machine keeps Objects in memory.

## Objects everywhere

In Ruby, everything is represented as an Object. Let’s consider a very simple object:

```rb
a = 3
```

The object `a` is an `Integer`, whose value is `3`. From that simple sentence, we can deduce that to keep that object in memory, the VM needs to know at least two things:

- The type of the object (it's an `Integer`)
- The object’s values (`3` in our case)

So how does the Virtual Machine keep this information in memory?

## Object Header

The way Objects are persisted in memory depends on the underlying processor architecture. In this blog post, we’ll only consider the x86_64 architecture.

As outlined above, the VM needs to hold in memory the object’s values and some metadata about it (like its class).

To keep track of the metadata part, the VM uses the [`RBasic`](https://github.com/ruby/ruby/blob/f3491042597ebdc3b93dc658f09ee6d260bc8592/include/ruby/internal/core/rbasic.h#L63-L86) C struct.

The `RBasic` structure is a 16-bytes header defined as follows:

| flags      | klass      |
|------------|------------|
| 8 bytes    | 8 bytes    |

`klass` is the reference to the Class of the current object. Remember that everything is an Object in Ruby, and a Class is also an Object. Thus in our example object `a`, its `klass` would be the reference over the `Integer` class object.

`flags` holds information about the type of object and other information useful for the garbage collector:

| Object type    | ruby_fl_type    | reserved      |
|----------------|-----------------|---------------|
| bits 0 to 5    | bits 5 to 30    | bits 31 to 63 |

Object type indicates the type of the Object (Class, Array, Hash, Float, etc.).
`ruby_fl_type` are flags used by the Ruby VM and the Garbage collector. For example, when an Object is frozen, the `RUBY_FL_FREEZE` flag is set.

### Heaps

To keep Objects in memory, Ruby uses 5 fixed-size slot heaps and one general heap.

If the Object is less than 640 bytes, then the object can be stored in one of the 5 fixed-size slot heaps. This is called *embedded* object allocation. The heap selected is the smallest one that can contain the object:

| Slot ID | Slot size |
|---------|-----------|
| 0       | 40 bytes  |
| 1       | 80 bytes  |
| 2       | 160 bytes |
| 3       | 320 bytes |
| 4       | 640 bytes |

If the object is larger than 640 bytes, then the object is kept in the general heap. This is called *extended* object allocation.

The benefit of using fixed-size slot heaps is to avoid memory fragmentation. When an object is freed, the slot can be reused by another object without fearing memory overlap. The general heap, on the other hand, will become fragmented over time and need to be compacted at some point to ensure further objects can still be kept in memory.

### Optimizations

Since the smallest slot size is 40 bytes, and the object header is only 16 bytes, it means there’s an additional 24 bytes free to be used by the object.

Each object type tries to use these 24 bytes to persist their data when possible. For example, the Array type will allow keeping 3 values in there:

| flags      | klass      | idx 0 value | idx 1 value | idx 2 value |
|------------|------------|-------------|-------------|-------------|
| 8 bytes    | 8 bytes    | 8 bytes     | 8 bytes     | 8 bytes     |

The astute reader might wonder where the length of that array is actually persisted. The answer is *using flags*! For Array, Ruby uses 7 of the `ruby_fl_type` flags to store the array’s length. Why 7 bits? Because with the largest fixed-size slot heap, we can embed up to 640 bytes, which means that we can store up to 78 values in such an array (640 - 16 bytes for the header divided by 8 bytes per value), and 78 can be represented using 7 bits.