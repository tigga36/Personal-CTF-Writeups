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

This requires an understanding of how stack works in binary. Recall that stack is an area of memory at the bottom. Execute `info proc mappings` to map out the memory. Looking at the bottom of the outputted map shows us that the stack is from *0xbffeb000* to *0xc0000000*.  
Note that the stack begins from the bottom, moving up as data is added. In this case, the bottom of the stack is at 8 bits before the *0xc0000000* end pointer, so *0xbffffff8*. Input `x/wx $esp` to examine the memory for $esp. Sure enough, the memory value for esp is the end of the stack buffer (approximately), *0xbffff79c* ***(Not sure why the stack pointer doesn't point to the exact address...)***  
  
Now, back to the instruction line. ebp refers to a *base pointer* and it contains an address pointing to somewhere in the stack. Whatever this address is, it seems pretty important, as it is being saved into the stack (which is basically being saved).  
  
The last instruction is:

>leave

Intel instruction reference tells us that *leave* is basically mov esp, ebp (set esp to ebp value), and pop ebp.   
***So, the instruction set is basically symmetrical.*** We first in the beginning push ebp, and set ebp to esp value and do the reverse in the end. This phenomenon is intuitive, as the entire binary must remember where to go back to in the binary after it is done executing the contents of the main section binary. The original 'next' instruction is saved in the stack, and all other information regarding the main function is stacked on top of that information during execution. To go back to the prior part of the binary, it pops the original location from the stack and goes to the given address.  
The next line,

>and esp, 0xfffffff0

which essentially ***masks*** esp and sets the last 4 bits to 0. This is done for formatting, and not essential in our solution. Next, the line is

>sub esp, 0x60

is subtracting a certain number from esp, having the stack pointer point a couple bytes lower than the base pointer. Then,

>mov DWORD PTR [esp+0x5c], 0x0

sets 0 to the stack with offset 0x5c which is what goes on for the `modified = 0;` code.  
  
***SO, THROUGH ALL OF THIS:*** The main binary moves the base pointer to where the stack pointer was referring to. Subtracting a certain number from the stack pointer moves it up, and creates a ***stack frame***, where local data of the main function can be handled (like our 0 variable). The reason the stack frame is so big is because we allocated a lot of space for it in the code.  
Instructions from there revolve around passing more parameters in the stack to call other functions, and a comparison, followed by a branch to handle each case.  

### Observing Execution Again/Automating Commands

Remove the currently-set breaks for main using `del`, and then set new break points at both before/after the gets command. This time, after setting the break points, we set automatic commands to execute on each break point. Input `define hook-stop` to initialize that, and set `info registers` to show registers, `x/24wx $esp` to show the stack, and `x/2i $eip` to print the next 2 instructions, and finish with `end`. After executing, continue with `c` and with it, input a lot of capital As. After inputting it, the break point after execution of gets will show the stack partially filled with *0x41414141*s, which are all ones that we just entered. Checking the value of memory where the 0 variable is stored with `x/wx $esp+0x5c`, we see that the value is still 0x00000000. We also see that there are 4 full rows that A needs to fill in order to reach that part, which is basically matches the 64 elements we need to exceed to input into the gets to over flow the 0 variable. Inputting that many characters seems that the 0 variable contents are changed.
