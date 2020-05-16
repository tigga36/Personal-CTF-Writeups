# Practice: Protostar Stack0

Link: https://exploit-exercises.lains.space/protostar/stack1/

So here's the second exercise after a week since doing the last one (I should dedicate a time of the week to this).


## The Problem

This question gives me the source code below:  

```C
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  if(argc == 1) {
      errx(1, "please specify an argument\n");
  }

  modified = 0;
  strcpy(buffer, argv[1]);

  if(modified == 0x61626364) {
      printf("you have correctly got the variable to the right value\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }
}
```

Looking at `volatile int modified`, it seems like compilation optimization is disabled, so this is like the last problem (modifying memory with overwrite), except this time its not usings `gets()`, but actually `argv`. So we can't use the gets vulnerability like last time. The difference between this and the last one besides that, is the use of `strcpy()` to transfer argv to a buffer. A quick google search revealed that `strcpy()` ***doesn't check for overflows in memory***. That is obviously a vulnerability we can abuse, and it seems like our tactic against `gets()` will work just as fine. Besides that, `buffer` seems to be the same size as the last problem. Inputting a string larger than 64 returns:

>Try again, you got 0x00000047

with different values for the 65th character. Furthermore, adding more characters seem to be increasing the number. My guess is this returns the correct answer with a certain input. 

## Similar Analysis from Earlier Problems

Instructions like `push ebp` seems to be resident, as its mostly the same function from last problem. Using `gdb`, we first set `break *main`, and use `set args` to set a 65-letter length argument. There seems to be a 

>cmp $0x61626364,%eax

Which obvivously seems like where are comparisons occuring there. We set a breakpoint there. Set automatic functinos to execute by using `define hook-stop` and setting `info registers` to see register info, `x/24wx $esp` to see the stack, and `x/2i $eip` to see the next 2 instructions. `run` and `c` a couple of times to run the program, and before the comparison, `w/wx $esp+0x5c` to check the contents of the memory whose values were moved to the `eax` register shows us that the value of the register is changed slightly, and that it is located at `0xbffff73c`, at the bottom of the stack memory.

So, we know that this is exactly the same problem we're dealing with here as the last one.

## Figuring out the arg

So the only thing left for us to do is to figure out the exact input to satisfy the compare condition. LiveOverflow gives us the hint that protostar is "little endian", as in the least significant bits are at the bytes with the lowest address value. As such, beyond the 64th value, the entered ASCII characters will fill up the memory starting from the least significant bit of the memory. Thus, entering `GF` after the 64th character of args will return us

>Try again, you got 0x00004647

with G corresponding to 47, and the F after that correspondign to 46. Following this pattern and the man page for ASCII, we can easily decide the necessary input to satisfy the buffer in being set as `0x61626364`. The resulting "flag" would become 

>dcba

Thus, inputting 

>./stack1 AAAAAAAAAABBBBBBBBBBCCCCCCCCCCDDDDDDDDDDEEEEEEEEEEFFFFFFFFFFGGGGdcba

gives us

>you have correctly got the variable to the right value

yay
