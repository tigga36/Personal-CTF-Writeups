# Practice: Protostar Stack0
  
Link: https://exploit-exercises.lains.space/protostar/stack0/

After a couple videos on LiveOverflow Binary course, I decided to try and solve this initial challenge myself because LiveOverflow told me to do so.  

## The Problem

This question gives me the source code below:  

```C
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  modified = 0;
  gets(buffer);

  if(modified != 0) {
      printf("you have changed the 'modified' variable\n");
  } else {
      printf("Try again?\n");
  }
}
```  
Which, when executed prompts an input. Typing something returns the "Try again?", so I just looked at the buffer, and assumed that this was a classic buffer overflow problem. So, when prompted, I inputted a string of length 65 with the tail element as a non-zero number. Sure enough, I got the output "you have changed the 'modified' variable" output. Yay?  

## Closer Look

I basically knew the answer to this question from the start, and just threw in a string. I think I could learn more from this, so I followed along with LiveOverflow's tutorial video on this problem to see the thought process.

### The code itself:

The only reason I thought this problem dealt with stack overflow was the fact that I saw a buffer of predetermined length. That usually won't be enough information for someone who doesn't know stack overflow, and that would most likely be me with any other concept right now. So, below are some pointers that LiveOverflow provided to analyze the given code.
- **volatile:** The `volatile` call to initialize the modified variable apparently disables the compilers optimization feature. If this wasn't in place, the compiler would notice that the modified varaible never changes, and omits the branch of when modified is nonzero in the executable binary. By disabling it with volatile, they allow this attack in the first place.
- **gets():** Now that I look at this part, I don't recall every using this command in C Programming classes. That's because if you check the man page for gets(), it tells you to **never use it***. Apparently it is impossible for gets to understand how many characters it will read, and will store values beyond the end of the buffer, allowing overwriting of existing memory. It is explicitly stated in the man page that ***it has been used to break computer security***. So that's an obvious vulnerability for me to manipulate for this problem.

### Using gdb:

Using gdb lets us look at the state of memories/registers etc. during the execution of a program.  
  
Launch gdb on the executable with `gdb ./stack0`. From here set a break point for gdb to pause execution to create a freeze frame of the state of the computer at a certain function. Input `break *main` to set a break point at the main function.  
  
`run` lets us execute the binary. The binary will pause execution at the breakpoint we set earlier, and we can execute commands again before resuming.  
  
`disassemble` allows us to dump the assembler code for the function we are at, main. `disassembler main` does the same thing. `set disassembly-flavor intel` lets us set the syntax of the assembler code to be in intel format, for preference. Execute `disassemble` again, and we get a view at the assembler code for main, the code we were given.  
  
There's a couple important ones, but lets look through all of it lmao:
The first instruction is

>push ebp  

This requires an understanding of how stack works in binary. Recall that stack is an area of memory at the bottom. Execute `info proc mappings`
