https://runestone.academy/ns/books/published/cpp4python/index.html
https://www.w3schools.com/cpp/cpp_function_overloading.asp

<img width="738" height="1212" alt="image" src="https://github.com/user-attachments/assets/91befaa1-1b91-46d1-9b8d-f24bd08dae3c" />


<img width="1000" height="500" alt="image" src="https://github.com/user-attachments/assets/22d13fd2-b5ed-4457-86be-bb06e02b4aee" />

https://www.brendangregg.com/FlameGraphs/memoryflamegraphs.html

https://rushter.com/blog/python-memory-managment/
https://cpp4arduino.com/2018/11/06/what-is-heap-fragmentation.html

- malloc()
- free()
- realloc()
- calloc()
- Heap memory vs. stack memory
- Heap fragmentation
- Virtual memory
- The Dangers of the Large Object Heap
- CPU cycles
- Cache trash
- Fragmentation and Garbage Collection
  
<img width="489" height="630" alt="image" src="https://github.com/user-attachments/assets/9211c157-bb29-4a4d-b20a-d6d444aec478" />

<img width="594" height="444" alt="image" src="https://github.com/user-attachments/assets/19156bc5-0809-48a9-90ea-2af36d29e6bd" />

Virtual address space is the maximum amount of address space available to an application, which depends on architecture, e.g. CPU register size. Virtual memory enables each process to have its own unique view of a computer's memory.

Physical memory is a storage hardware, made up of physical memory devices, which is organized as an array of M contiguous byte-sized cells. Each byte has a unique physical address (PA).

Physical memory addresses are unique in the system, virtual memory addresses are unique per-process.

Only the kernel uses physical memory addresses directly. Userspace programs exculsively use virtual addresses. Translation from virtual to physical address needs the combination of OS software, address translation hardware in MMU, and page table stored in physical memory or disk.

<img width="1523" height="641" alt="image" src="https://github.com/user-attachments/assets/26785deb-deca-428f-8ba5-497792623491" />

https://witscad.com/course/computer-architecture/chapter/virtual-memory

<img width="600" height="449" alt="image" src="https://github.com/user-attachments/assets/9e4960e5-39b0-49da-bd43-48e80765d935" />

https://www.insidetheiot.com/fragmentation-and-garbage-collection/
https://gabrieletolomei.wordpress.com/miscellanea/operating-systems/virtual-memory-paging-and-swapping/



- Heap map — a picture of used/free blocks on the heap (where dynamic objects live).
- Stack — per-thread LIFO space for function frames, locals, return addresses (fast, short-lived).
- Virtual memory — each process’s address space mapping virtual addresses to physical pages; gives isolation and illusion of big RAM.
- Physical memory (RAM) — actual hardware memory; pages from virtual memory map here.
- Arena / pool / block (CPython pymalloc) — CPython groups small-object allocations: arenas (~256 KiB) contain pools (~4 KiB); pools serve fixed-size blocks (size classes up to ~512 bytes). Objects >512B go to system malloc.
- Memory leak — memory that’s still reachable (or not reclaimed) even though program no longer needs it; in Python often due to lingering references or extension/C-level leaks.
- Virtual cache (page cache) — OS caches file/IO pages in RAM to speed future reads (not CPU cache).
- Page tables — OS structures that map virtual pages → physical frames.
- TLB (Translation Lookaside Buffer) — tiny CPU cache of recent virtual→physical page mappings; TLB misses are costly.
- Swapping — moving pages between RAM and disk (swap); heavy swapping = massive slowdowns.
- Address space — the full set of virtual addresses a process can use (separate per process).

Pointers are exactly what they sound like, if you have encountered ‘References’ a pointer is very similar. A pointer is literally the memory address of a piece of data, the type you assign to the pointer dictates what you expect to find at that address (such as a class, an array, or an integer).

You use a pointer typically to avoid moving around big chunks of memory. If you imagine the basic robot class, it has a large number of variables for motors drive systems etc, all of these are stored in memory. If you call a funtion and pass in the ACTUAL robot, you end up making a copy of the whole robot (remember if you change a variable you pass into a function, you are changing a copy, not the original) which can be a lot of memory to copy and is quite wasteful. If you simply pass a pointer, you essentially say “our robot is located a memory address 0x800f000” and that pointer (the address only) becomes the variable which is copied into memory, when you look up that memory address you get the real robot, so if you change attributes of the robot, you are not changing a copy, but rather the real robot.

A semi-similar example would be if you were trying to send someone a file you downloaded from the internet, you could either email the file itself (giving them a copy of the same file) or email them a link to the file. In both cases they have access to the file, but the latter is more efficient.
 
Pointers are valuable because they allow us to increase program execution speed. This results in simpler code, especially as the size of our code increases.

For example, we can use pointer variables to modify and return by function, a much better alternative to passing-by-value. Without pointers, we would have to manually copy values, thus ending up with bulkier code. This works fine when working with a single integer but can be more complex when working with a more advanced data structure. Let’s take a look at an example
 
Unique pointers: allow only one owner of the pointer, and does not allow copying. Once this one owner goes out of scope, the pointer is cleaned up

Shared pointers: allow multiple owners, and keep track of the number of owners it has. Once that reaches 0, they clean themselves and their underlying memory by themselves

Weak pointers: usually used in conjunction with smart pointers, and allow for retrieving a reference to a smart pointer without incrementing the smart pointer's owner (reference) count
