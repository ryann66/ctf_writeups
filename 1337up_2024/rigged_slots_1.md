# Rigged Slots 1
I started out by analyzing the decompiled code in Ghidra.  The program would reveal the flag if you successfully managed to accrue 133742 dollars using your starting balance of 100 dollars.  The gambling mechanism was implemented using the C standard library `rand()` function, which is only psuedorandom.  
The program used the common method of seeding `rand()` with `srand(time(null))`, which uses the time in seconds as a random seed. However, since `time(null)` only works at second granularity, I could easily start a second random number generator with the same seed, and analyze its results.  This allows me to predict the outcome of the server's random number generator before I input my bet.  Using this, I can make money off the game.
The gambling program also imposed a time limit of 3 minutes, which required a faster rate of gambling then I could do by hand, inputting numbers into two parallel programs. This meant I needed to write a program to manage the gambling for me.

The program starts sends a request to start gambling to the server and starts a local instance of the gambling program.  Then, after clearing the welcome message from the gambling program, it enters an infinite loop where it scrapes the result of betting $1 locally and uses the outcome to bet either the maximum it can afford or $1 with the server.

```python3
#! /bin/python3

from pwn import *

tester = process('./rigged_slot1')
winner = remote('riggedslot1.ctf.intigriti.io', 1337)

tester.recv(1024)
print(winner.recv(1024).decode())
bal = 100

while True:
    tester.sendline(b'1')
    s = tester.recv(1024).decode()

    if 'lost' in s:
        winner.sendline(b'1')
        bal -= 1
    else:
        winner.sendline(str(min(bal, 100)).encode())
        bal *= 2

    print(winner.recv(1024).decode(), end=None)
```

Finally, we track an approximation of the balance that we have on the server. Since we're only allowed to bet $100, this doesn't have to be very accurate.  So we assume that, in the first 50 bets, we're going to win at least one.  We also assume that we will never drop below $100 once we get above it.  These approximations work well enough to bootload the program to the point where we can always bet $100 without worrying that we're betting more than our balance.
