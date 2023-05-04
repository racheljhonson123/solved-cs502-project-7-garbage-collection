Download Link: https://assignmentchef.com/product/solved-cs502-project-7-garbage-collection
<br>
<h3>ASM Lowering and Virtual Machine</h3>

The skeleton code you are given for this assignment contains two elements:

The compiler, with reference implementations of the phases so far (but no Optimizer), a CPS register allocator and a CPS to assembly language transformer in compiler/src/miniscala/CPSToASMTranslator.scala;

A virtual machine for executing the assembly code, in the <sub>vm </sub>directory.

The virtual machine contains two main components:

The interpreter, located in vm/src/engine.c (and with the header le vm/src/engine.h);

The memory system, implemented in vm/src/memory_nofree.c and vm/src/memory_mark_n_sweep.c (and with the header le <sub>vm/src/memory.h</sub>).

The virtual machine also contains basic infrastructure like a make le and 4 tests you can use to check your GC implementation.

<h3>Getting Started</h3>

To start, unpack the skeleton code as you have done for the last assignments. Now, you will have the <sub>vm </sub>directory. Go there and run <sub>make:</sub>

$ cd vm/

$ make

Use the following targets:

<ul>

 <li>‘make vm’ to use your mark-sweep GC</li>

 <li>‘make test’ to test the VM</li>

 <li>‘make clean’ to clean the VM</li>

</ul>

$ make test rm -rf bin mkdir -p bin

cc -std=c99 -g -Wall -O3  src/engine.c src/fail.c src/loader.c src/main.c src/memory.c -o bin/vm

Queens test failed!

Bignums test failed!

Pascal test failed! Maze test failed!

You can also run the tests individually, since they’re included in <sub>examples:</sub>:

$ bin/vm ../examples/asm/queens.asm enter size (0 to exit)&gt; 10

Error: no memory left (block of size 2 requested)

Your task is simple: make the tests pass by implementing a mark and sweep garbage collector instead of the no-GC memory system currently implemented in <sub>vm/src/memory_nofree.c.</sub> In order to test the vm with your garbage collector, you will need to change the value of the variable GC_VERSION in the le <sub>memory.h</sub>.

Okay, that’s it with the intro. Now for real work.

<h2>The Memory</h2>

We rst brei y describe how the VM engine interacts with the memory manager and subsequently detail the implementation that is required for this assignement.  When the virtual machine is started, the rst call is to memory_setup.  This allows the memory manager to allocate the total memory (the amount can be changed using the -m &lt;size&gt; command line option of the vm). The “total_size” parameter passed to this method indicates the total number of <strong>bytes</strong> needed (the default value is 1000000). The method memory_get_start returns a pointer to the beginning of this allocated memory. The total memory is used to store the program code and the heap.

The virtual machine then loads the program code into the memory. Afterwards, the method memory_set_heap_start is called, indicating the rst address that directly follows the program code. The memory starting from this address can be used by the memory manager to store its data structures and for allocating blocks in the heap:

The function memory_allocate is invoked when a heap block needs to be allocated.

<h3>Virtual and Physical Addresses</h3>

The memory can be seen as an array of 32-bit values referred to as <strong>words</strong>, which is represented by the type value_t in the VM. Every entry in the array can contain arbitrary 32 bit information: tagged values, virtual addresses (explained below), block headers or registers (which are also stored in the heap)

There are two kinds of addresses in the VM:

Physical addresses, represented as values of type value_t*, are pointers to the elements of the memory array.

Virtual addresses, represented as values of type value_t, are relative to the starting (physical) address of the memory and only point to individual <strong>words</strong>, so their last two bits are always 0. This satis es the tagging requirement for references, which states that they should end in bits 00;

The pointers stored in the heap are always virtual addresses. The addr_v_to_p and addr_p_to_v methods in the le engine.c convert between the two address types.

While physical addresses are addresses of words in the memory, virtual addresses are virtual machine pointers. <strong>It is essential that addresses stored in the heap are virtual addresses and not physical addresses (i.e, pointers to the elements of the memory array)!</strong>

For ef ciency reasons, the pointers stored in the base registers (result of engine_get_base_register) are physical addresses.

The <strong>size</strong> parameter passed to the memory_allocate function is numbers of <strong>words</strong> that have to be allocated.

<h2>The Garbage Collector</h2>

You are required to write a memory manager that implements the interfaces de ned in the “memory.h” le and performs mark-sweep garbage collection. Below we provide an overview of the data structures that have to be implemented and also a few tips and tricks.

<h3>Block Headers</h3>

As discussed in the lectures, memory is allocated and freed in chucks of words referred to as blocks. Each block has a tag and a size which are passed as parameters to the “block-alloc” primitive. We refer to the starting address of a block as a “block pointer”. Use the rst word of a block to store the tag and size of the block (which are referred to as block headers). Therefore, to allocate a block of size n you will need n+1 words.

<h3>Free list</h3>

You GC must have a (singly-linked) free list containing all the free blocks. The second word of a free block can be used to store the address of the next element of the free list.

There is a slight trick with free lists. A free list entry contains at least 2 words. But the library allocates blocks of size 0 (and thus 1 word) that your GC can later free. This produces 1-word entries in the free list, which won’t work. To overcome this, the easiest solution is to allocated at least two words, even for blocks of size 0.

The free list is used to allocate blocks. When having multiple free list entries, you should either pick the smallest one that ts the necessary size (best t strategy) or the rst one that ts ( rst t strategy).

<h3>Segregated free lists</h3>

Instead of having a single free list, it is much more ef cient to have several free lists, one per block size up to a maximum block size, plus one free list for the bigger blocks. That way, in many cases, no iteration is necessary to nd a free block of a given size.

We suggest you write a rst version of your GC with a single free list, and once it works, you update it to have 32 segregated free lists. As described below, only a GC using segregated free lists could get you the maximum number of points for this assignment.

<h3>Pointer Bitmap</h3>

You need to maintain a bitmap with one bit per valid heap address. This bitmap is used for two purposes:

<ul>

 <li>to determine if a value stored in the heap is a valid block pointer. Even though we use tagged values, it ispossible that, during an arithmetic operation, an untagged value is left in one of the registers. Since the registers of our virtual machine are also stored in the heap, we may mistake an untagged value for a virtual address in the marking phase. So although it looks like a virtual address, the untagged value may point anywhere, including invalid locations and in the middle of blocks. In order to prevent the GC from following incorrect addresses, we mark the beginning of each block in the bitmap.</li>

 <li>during marking, we reset the bit to mark a block; therefore, at the end of the marking phase, the markedblocks will be those whose bit is not set in the bitmap. During the sweeping phase, you will also have to coalesce successive entries in the free list, so the heap memory does not get too fragmented. Also don’t forget to update the bitmap so that allocated blocks have their bit set, and free blocks have their bit cleared.</li>

</ul>

To complete the above picture, the memory layout would actually look like this:

<h2>Testing</h2>

You can either use the make test command to execute the tests automatically, or you can run each individual test by hand (good for debugging):

$ bin/vm ../examples/asm/queens.asm enter size (0 to exit)&gt; 10

Error: no memory left (block of size 16 requested)

The expected sizes your program should run on are:

for queens.asm: without GC – breaks at 8, GC works with 15 for bignums.asm: without GC – breaks at 214, GC works with 1000 for pascal.asm: without GC – breaks at 57, GC works with 200 for maze.asm: without GC – breaks at 7, GC works with 20

The test given to you do not check for the correctness of the output, it only check that the program run without Out Of Memory errors.

If you want to test simpler program, you can compile them with the compiler rst: sbt “run file.scala”. It will generate the le out.asm in the compiler folder. You can then run the vm on it.

<h2>Debugging</h2>

When debugging you may want to trigger a garbage collection early on. To do so, print the code size and adjust the memory with bin/vm -m &lt;bytes&gt;  such that the size is just above the code size + twice the bitmap size. This will let you trigger the garbage collection early, when the program hasn’t yet allocated too much space, allowing easier debugging.

<strong>Implement a procedure to check the data strcuture invariants</strong>

For ease of debugging, it is recommended that you implement a procedure that traverses every block in the heap and checks for the correctness of the headers and the freelist. To traverse all blocks in the heap you can start from the rst block and use the block size (stored in the header) to jump to the start of the next block and so on. Also, use “assert”s (de ned in assert.h header le) in all places in the code where you think an invariant must hold. This will greatly help in debugging. Of couse, this is only recommended and not mandatory