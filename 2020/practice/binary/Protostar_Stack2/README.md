# Practice: Protostar Stack2
  
Link: https://exploit-exercises.lains.space/protostar/stack2/

The problem gives us the source code below. Apparently it "looks at how environment variables can be set".

```C
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  char *variable;

  variable = getenv("GREENIE");

  if(variable == NULL) {
      errx(1, "please set the GREENIE environment variable\n");
  }

  modified = 0;

  strcpy(buffer, variable);

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }

}
```

This program apparently wants us to set an environment variable by the name `GREENIE`, to a certain value `0x0d0a0d0a`. This code also seems to be using `strcpy`, suggesting that we could use overflow to overwrite `modified` by some value. 

Upon closer look, it seems like the rest of the problem beyond the environment variable is the same problem as stack1. 
