---
title: "Windows: WSL Syscall Working"
date: 2018-09-06

---
# Decoding WSL Syscall from the kernel

WSL is implemented in Windows using picoprocess concepts.  Picoprosses get separate syscall dispatcher.
WSL Core implementation is provided by two Windows drivers
* **LXCORE**
Provides implementation of Linux Syscalls
* **LXSS**
LXSS loads LXCORE and initializes the driver

The Syscall is made by the user process with the convention explected by the picodriver. The call stack of the syscall is provided below. 

## Call stack of a Syscall

```
Child-SP          RetAddr           Call Site
ffffc705`413649c8 fffff803`1c50e82e LXCORE!LxpSyscall_GETCWD
ffffc705`413649d0 fffff803`1c52a766 LXCORE!LxpSysDispatch+0x172
ffffc705`41364aa0 fffff802`e6f151ff LXCORE!PicoSystemCallDispatch+0x16
ffffc705`41364ad0 fffff802`e6972b52 nt!PsPicoSystemCallDispatch+0x1f
ffffc705`41364b00 00007ffe`1b527b74 nt!KiSystemServiceUser+0x82
00007fff`c5c139a0 00007ffe`1bb60000 0x00007ffe`1b527b74
00007fff`c5c139a8 0000ffff`ff00ffff 0x00007ffe`1bb60000
00007fff`c5c139b0 00000000`00000000 0x0000ffff`ff00ffff
```

## nt!KiSystemService

KiSystemService is the kernel functio providing system services. This function is triggered after a service request is called. It decides btween 32bit and 64bit call and jumps to appropriate functions. Since picoprosses are 64-bit prosses, it will jump to function 'nt!KiSystemServiceUser'

## nt!KiSystemServiceUser

The function `nt!KiSystemServiceUser` function determines if the Syscall is initiated by PicoProcess (WSL). It checks if the process is minimal process from the flag at 0x3 location of _KTHREAD structure. 

![1.PNG](attachments\7aba899d.PNG)

ebx contains pointer to _KTHREAD structure.

![minimal flag.png](attachments\1fc1113a.png)

If the minimal flag is set in the KTHREAD, it calls `nt!PicoSystemCallDispatch`
## nt!PicoSystemCallDispatch

![2.PNG](attachments\fcab642b.PNG)
`nt!PsPicoSystemCallDispatch` function is a wrapper. It copies the address of `LXCORE!PicoSystemCallDispatch` function to eax and calls eax transferring control to LXCORE

## LXCORE!PicoSystemCallDispatch

![lxcore!PicoSystemCallDispatch.png](attachments\c523ba9f.png)

This function passes the controll to `LxpSysDispatch` after incrementing the value of `LxpSystemCallCount`. 
 
## Lxcore!LxpSysDispatch
With syscall number in eax it calculates the address of the the function which it has to call and based on the number of parameter size it transfers control to the calculated address. 


 [rcx] is copied to rax

```
mov     r13, [rax+30h]      //syscall number
mov     rcx, [rax+148h]     //parameters
mov     rdx, [rax+150h]     //parameters
mov     r8, [rax+40h]       //parameters
mov     r9, [rax+58h]       //parameters
mov     r10, [rax+48h]      //parameters
mov     r11, [rax+50h]       //parameters
```
