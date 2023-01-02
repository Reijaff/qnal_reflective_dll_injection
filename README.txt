# qnal_reflective_dll_injection
https://www.youtube.com/watch?v=IX0qUTbXNog

Script

Reflective dll is just shellcode for lazy people .
Why bother to rewrite all your code to be position independent, when you can just write one function, that's gonna resolve your portable executable, preserving all the offsets, to structures, functions and libraries.

The most basic way to inject dll into another process, is to execute load library function inside target process, with name of the dll as an argument.


Get a handle to target process. Allocate memory and write path to dll string inside target process. Get load library function address inside kernel dll virtual memory. Create remote thread executing load library function inside target process. This injects our dll in target process memory, and executes Dll main function.

Dynamic library kernel32 dll almost always loads at the same virtual address in every process. so by finding address in one process, we know addresses in all processes.

Final execution. inject dll. check if dll present in target process modules.


But, this method, does not meet our needs. Reflective loader must not interact with files on disk . so, load library function is not sufficient. It cannot resolve a dll that happens to be in virtual memory.

We need to rewrite load library function from scratch, and make it work with files in virtual memory.

Next, make it position independent, and put it inside dll file .

Now, if we write this dll into target process memory, and use this reflective loader as an entry point of execution, it will resolve it's own image, it's sections, dependencies, and relocations.
then, execute the code in main function, that is, non-position independent code.


Review the code that injects dll file into target process. At this stage, dll may look like this. When compiled , there is a reflective loader function placeholder as export .

Load dll file into virtual memory. find the offset to the reflective loader function. Get handle to target process. Allocate memory for dll file in target process in read-write mode. Write dll file into allocated memory. Change memory mode to read-execute . Calculate address of reflective loader function. Create remote thread executing reflective loader function in target process.

We should go back and look at this stage in code more closely, to answer the question. How to find reflective loader function offset in dll file that isn't loaded.


Resolved, portable executable in virtual memory. Portable executable on disk. One of the main difference, is. Raw offset, to each section in memory.

In resolved, portable executable it's called relative virtual address.
in file, it's just offset .

Our goal is to find export function, in file. But we only know how to find export function via relative virtual address. We cannot use relative virtual address on file directly. So we need a way to convert it into offset, and calculate a pointer .

Let's assume that we want to calculate off set to address in section 2 .

Let's assume the following relative virtual address. First we need to find what section it points to. Then we calculate offset within the section. Get offset to that section in file and add offset within the section.
Calculate the address.

function aR V Ay to offset, that loops through each section header, and calculates raw offset .

Back to finding export address. Standart way of searching for export function through Export Address Table. But, every time we get relative address, we convert it to offset.

Finally, using offset to calculate function virtual address in target process.

Running the injector under a dbugger.
search for a main function.
search for a line where create remote thread is called.
put a breakpoint there.
Run the process until the breakpoint .
Target process is created and dll file is written in it's memory. Open target process with a dbugger.
Address of reflective loader function in target process virtual memory.
put a breakpoint there.
Switch to Injector process dbugger, and continue the execution.
in target process dbugger, execution stopped at breakpoint.
in a thread that got created on reflective loader address.

Now , we need to write reflective loader position independent function inside dll.

Step 0 in reflective loader is to get it's own position in virtual memory. then, to find address, of it's own image.

Moving down from found position until first portable executable magic number is reached.

step 1, is to get addresses to necessary system functions in libraries, that are, already loaded in target process.
Do it manually, parsing process environment block. Then, iterate over export functions, and get their addresses.
All auxiliary functions must be implemented manually. They must have, inline prefix .

When the inline function is called, whole code of the inline function gets inserted or substituted, at the point of inline function call.

Names of functions must be allocated, as stack strings.

Step 2, is to allocate memory for loading a dll. read-write-execute mode.

Step 3, is to copy headers of image to allocated memory.

Step 4, is to copy all sections one by one, to their offsets in allocated memory.

Step 5, is to resolve, import - address table.

Import-address-table is a structure that contains pointers to offsets, to names, of import functions. To resolve an import function, means, to overwrite offset to function name, with an actual address, to a function.

Use previously resolved, load library function to load library dependencies into process memory. Then, use get proc address function to get addresses of functions in those libraries.

And overwrite function name offsets with them.

Step 6 , is to process all relocations.

Relocations table is a structure that contains relative virtual addresses to sections in which to perform relocations. for every relocation there is an entry that consists of type of relocation, and an offset to each section relative virtual address .

Calculate address, where to perform relocation. Base virtual address, plus relative virtual address of relocation block, plus relocation offset.

How to perform a relocation?

every relocation address points to a dummy virtual address.
we need to change, dummy virtual address, to correct virtual address.
a dummy virtual address consists of dummy image base, plus relative virtual address.
To calculate correct virtual address, we need to add base virtual address, to relative virtual address.
Then, overwrite, dummy virtual address.

process every entry according to it's type.

At this point dll has been resolved. calculate entry point address, and call it like a function.

dll main function poc !

Execution proceeds, at the entry point of loaded dll. until dll main function.

Running the injector. If injection succeeded, we see Message box executed from main dll function.

the results. code for executable file that injects dll into process reflectively, and, code for dll file that can be loaded reflectively.
