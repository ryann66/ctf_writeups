# Comma Club
I started by testing out various inputs for the voter count.  This led me to discover that, while the number of votes that any one candidate could get was capped at the population of the state of Wyoming, the combined total was never checked. Furthermore, I discovered that adding the vote count `583100` for both candidates led to a illegal instruction signal being raised.  This obviously signalled that some form of vulnerability had been found.  Furthermore, I now know that either the program has tried to execute data or that the program has jumped a number of bytes specified by my inputs.  Testing this, I ran the program until crash in GDB, which revealed that execution had failed inside the `vote_printer_selector()` function.
```gdb
Program received signal SIGILL, Illegal instruction.
0x0000555555555430 in vote_printer_selector ()
```
Disassembly of the vote_printer_selector() function reveals the cause of the SIGILL.
```asm
0x0000555555555400 <+0>:     endbr64
0x0000555555555404 <+4>:     push   %rbp
0x0000555555555405 <+5>:     mov    %rsp,%rbp
0x0000555555555408 <+8>:     sub    $0x10,%rsp
0x000055555555540c <+12>:    mov    %rdi,-0x8(%rbp)
0x0000555555555410 <+16>:    mov    -0x8(%rbp),%rax
0x0000555555555414 <+20>:    movzbl 0x20(%rax),%eax
0x0000555555555418 <+24>:    cmp    $0x20,%al
0x000055555555541a <+26>:    jne    0x55555555542a <vote_printer_selector+42>
0x000055555555541c <+28>:    mov    -0x8(%rbp),%rax
0x0000555555555420 <+32>:    mov    %rax,%rdi
0x0000555555555423 <+35>:    call   0x5555555556ae <total_vote_printer>
0x0000555555555428 <+40>:    jmp    0x555555555436 <vote_printer_selector+54>
0x000055555555542a <+42>:    mov    -0x8(%rbp),%rax
0x000055555555542e <+46>:    mov    %rax,%rdi
0x0000555555555431 <+49>:    call   0x5555555554ac <simple_vote_printer>
0x0000555555555436 <+54>:    nop
0x0000555555555437 <+55>:    leave
0x0000555555555438 <+56>:    ret
```
The program has, for some reason, jumped into the middle of an instruction, causing it to be misinterpreted as an illegal instruction.

Next, I test how the vote count might modify this jump. I test the vote counts `583100` and `583101`, finding that it jumps to the address 1 greater than the previous address.  This address is a valid instruction, the call statement to `simple_vote_printer()`. This suggests that a linear change to the total of the two vote counts will result in a linear change in the address jumped to.  Using this, I attempt to jump to the `close_voting()` function, bypassing the normal password check.  However, when I try this, I find that the program instead jumps back into the `vote_printer_selector()` function.  This suggests that the program has an off by one bug, and that only one byte of the address it jumps to can be changed.

Fortunately, the next function in memory is the `change_password_to()` function.  This function accepts a string as an argument and does not check that the user knows the password.  So, by jumping directly to the function, I can change the password to whatever was meant to be passed in as an argument to `vote_printer_selector()`.  This appears to be some form of struct type containing many different fields.  However, the first field is the candidate name.  In this case, "Total". Since this call was made 'legitimately' by a C function pointer-not some ROP scheme-the stack remains uncorrupted.  Therefore, the program continues on, printing the vote counts for the two candidates as normal before returning to the main menu.

At this point, I know the password and can close the polls.  This returns us to the shell, where I run the `ls` command, revelaing the file containing the flag, which I then use the `cat` command to retrieve.
