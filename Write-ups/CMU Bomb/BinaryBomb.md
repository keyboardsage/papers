#Bomb Write-up
##Introduction
The Carnegie-Mellon Bomb Lab is a fictional bomb created by Dr. Evil. The purpose of the Bomb Lab executable is to better your understanding of 32-bit x86 assembly and the Gnu debugger. This document is a brief write-up that you can use while attempting to reverse engineer the bomb.

Contained within are hints and below them the solutions. Do not scroll down to the solutions until you are ready.

##Technical Information
The download link to the bomb executable used for this write-up is located [here](http://www.opensecuritytraining.info/IntroX86_files/bomb.tar), it is the one from [OpenSecurityTraining's IntroX86 page](http://www.opensecuritytraining.info/IntroX86.html). For the adventurous that enjoy these type of low-level lab assignments visit this [CMU lab page](http://csapp.cs.cmu.edu/2e/labs.html) for more system's engineering type challenges.

**Filename:** bomb.tar

**MD5 Hash:** c61c4192e429a424a12e0814f22b419b

**SHA-1 Hash:** 755aeab8bc055e1451d7f36d3246f38fa2074f5a

**SHA-256 Hash:** cc44ccb32b6cc74d76b43e5ee791296a9f351b23f07623768af1aa1da6b4711a

**SSDEEP Hash:** 384:3XMsnOdrcDXbGKmRVZu3v3Zf15v2/EEtfOaKP+VIWzNvjp7:Ml4UVZav3Zf15v2hfOgVIWZvjp

##Hint
###Phase 1
Make sure you understand calling conventions and how the stack works.
###Phase 2
Baby steps, understand line by line what is happening. Leave no stone (or function) unturned.
###Phase 3
Sometimes in chaos it is best to choose a path.
###Phase 4
Multiplying like rabbits.
###Phase 5
Standing on the shoulders of ...
###Phase 6
All data has a structure.
###Secret Phase
Secrets stay secret.

##Solution
###Phase 1
The return value of *strings_not_equal* is used in the comparison that determines whther or not to explode the bomb. Right before the call to *strings_not_equal* two addresses are pushed onto the stack. Examine those addresses using `x/1bs 0x80497c0` and `x/1bs $eax` so gdb will display the locations being referenced as strings. You see your input and then an unknown string the function uses as a comparison. So the answer is "Public speaking is very easy."

Beyond the solution, what *strings_not_equal* did is compare your input's string length vs the answer's string length. Following that it compared each character in the strings one by one to ensure they were equal. If you failed either of those checks the bomb would explode.
###Phase 2
The phase calls a function named *read_six_numbers*. Inside the function *read_six_numbers* you see a call to the C function sscanf. Now remember parameters are pushed onto the stack in reverse order; in other words the last argument is pushed onto the stack first. That means sscanf's format string is the second to last argument pushed onto the stack before the sscanf call. Examining that address with `x/1bs 0x8049b1b` reveals "%d %d %d %d %d %d". So now you know the format of the text you should input. The 6 pushes that reference edx are going to be used as locations to save your input values. So in a nutshell this function just saves the 6 numbers you input to the stack side-by-side (as an array). Run `finish` and return back to the caller function *phase_2*.

While continuing through the code we see 3 things as follows:

1. Eventually this little gem comes up, `imul eax,DWORD PTR [esi+ebx*4-0x4]`. The first time through ebx is 1 so the statement evaluates to esi+4-4, or simpler put esi itself. So all this does is multiply eax by the integer 1 and save it back into eax; in other words it appears to act like a NOP.
2. The following instruction seems to explode the bomb if the calculated eax value doesn't match your first input value that you now have saved on the stack.
3. Then after that the increment, compare, and jump sequence of instructions is a dead giveaway that we are dealing with a loop. We know it is a loop and not a conditional statement because the address we jump to is earlier in the code.

Bullet point 3 above is the eureka moment because you know that the ebx is not always 1. That means the ebx register gets incremented and used to generate values for eax at bullet point 1. Subsequently that means bullet point 2 compares those values to your input to make sure they are the same. Run `break *0x8048b7e` and now you can read the value from the eax register just before it is compared to what you entered. These values represent the values you **should** have entered. So the answer is "1 2 6 24 120 720".
###Phase 3
Like in the previous phase your eyes can't help but lock in on the sscanf function. The procedure for deciphering is still the same. When you look at the second to last parameter on the stack with `x/1bs 0x80497de` you get the format string, "%d %c %d".

The instructions following do some sanity checks:

- Making sure you input 3 pieces of data)
- Assert the first input value is less than or equal to 7

Ultimately the code jumps to the location stored inside address eax*4+0x80497e8; eax being the first value you entered. Running `x/1wx (0x80497e8+4)` simulates what would happen if the integer 1 was entered as the first input value. The output of that simulation is 0x08048c00 so now we know where we will jump too.

Going to that location shows the value 0x62 saved to register bl, this will become important later. The next 2 instructions compare 0xd6 to our third input value and jump if they are equal so this must be our 3rd value.

Now if we were to enter in the correct 1st and 2nd values we would jump so lets read the code to see what would happen after the jump. Remember the value stored in register bl? That value is used in a comparison so this must be the character we were supposed to enter as the second input.

Convert third input 0xd6 to a number and second input 0x62 to an ascii character and you have the answer. So the answer is "1 b 214". Keep in mind though that this is only one of many possible answers. There are other input values we could have used to satisfy the comparisons and that would have taken us down alternate paths of execution.
###Phase 4
The function this phase used the sscanf function yet again. Run `x/1bs 0x8049808` and find the format string, "%d". Now we know that the format of our input should be a number.

Sometime later there is a call to func4. Who knows what func4 does but below it after func4 the remaining instructions indicate eax needs to be 0x37, decimal number 55. So now we know whatever gets entered must cause func4 to set eax to 0x37.

Now we know skipping over evaluating func4 isn't good enough, time to evaluate it. It looks a little daunting but what was immediately obvious is that it is a recursive function because it calls itself. Additionally, a block of instructions appears twice (add, lea, push, call, mov). The instructions are almost identical except the first call decrements ebx by 1 while the second decrements it by 2. At this point all the programmers in the room are probably thinking the same thing; a function that takes in only 1 number, is recursive, and does a decrement by both 1 and 2 when making calls...it is probably one of the most popular computer science functions taught, the fibonacci sequence. Following every line of execution would take to long though, so setup a breakpoint at 0x8048d1d and run the program with different numbers to see what eax evaluates to after func4. I used 5 and 7 as input and the function returned 8 and 21 respectively. This confirms that this is a fibonacci sequence. Naturally, this means you need to input 9 in order to get the result 55. So the answer is "9".

If you don't have a programming background maybe the fibacci sequence would not have jumped to mind but if you treat the function as a black box and do black box testing you could have figured the pattern out and determined you needed to input 9.
###Phase 5
Some quick static analysis shows that the string entered needs to have a length equal to 6 otherwise the bomb explodes. The comparison at 0x8048d43 gives that away.

The phase zeros out edx, saves the address of your input in ecx, and saves 0x804b220 in esi. Then a loop begins on 0x8048d57. You can tell the loop begins there because 0x8048d69 jumps backward in the program to 0x8048d57. The inc, cmp, jle confirms it loops 6 times. The first two instructions `mov    al,BYTE PTR [edx+ebx*1]` and `and    al,0xf` just put a byte of your input in eax and throws away everything but the lower nibble. Then that number is used to index the string at 0x804b220; run `x/1bs 0x804b220` and you will see that string is "isrveawhobpnutfg\260\001". You may be wondering why the \260\001 is present, it is because it isn't actually a string at all but rather it is an array that is being used as a lookup table.

Later in the function one of the parameters to *strings_not_equal* is 0x804980b and the string at that location is "giants".

So now there is a goal, input values that will cause the loop to index the right letters so we can spell out the word "giants". That means we need the numbers 15, 0, 5, 11, 13, 1 in the lower nibbles. We can enter any ascii characters we want to accomplish this, I chose O05KM1. So the answer is "O05KM1" but it is only one of many possible answers.
###Phase 6
This one is a little more involved. You see the usual *read_six_numbers* which you encountered previously so out the gate you know the format of the needed input. The eax register was loaded with ebp-0x18 so that is the address of where your input gets saved in memory. Inspect it using `x/10wx $ebp-0x18` for a reminder of how it gets saved on the stack.

As a I mentioned previously this phase is a little involved so I'm going to refer to each of the obstacles we face as hurdles and below I have listed each hurdle using bullet points so that the information is easier to digest.

- Hurdle 1: Trace until the first comparison at 0x8048dc7. It appears that all it does is assert that the first input value is less than or equal to 5. However, don't be mislead and miss the subtle `dec    eax` nuance here; that means this comparison really checks to see if your first input's value is less than or equal to 6, not 5.
- Hurdle 2: Keep going until the second comparison at 0x8048dd4. There is no apparent way to take the branch because ebx will always be 1 at this point and 1 is not greater than 5. If we could take it though we would jump just pass the end of hurdle 5.
- Hurdle 3: First time through reaching the next comparison at 0x8048dec the ebx register will be set to 1 as a side effect of hurdle 2. Therefore, your first input value gets compared to your second one. We can only progress if they are not the same value. The moral of the story here is the first and second values can't be equal.
- Hurdle 4: So now moving on to 0x8048df7 something interesting happens. The comparison is a conditional for a loop because it jumps to an earlier address, 0x8048de6. Just a moment ago it appeared that the first and second input values needed to be unique but now due to this loop ebx will be altered; the effect of that is the first input value needs to be unique from **all** the other input values.
- Hurdle 5: Moving on we run into another comparison at 0x8048dfd. It is associated with yet another loop! The jump back engulfs our previous loop so now we know that we are dealing with a nested loop. This means we will repeat the previous hurdles **but** this time due to edi being incremented a different input value gets used as a guinea pig each time through. So for instance in hurdle 1 we are now checking that **all** input values are less than or equal to 6. At hurdle 4 it is not about your first input value being unique, they **all** must be unique. I think you get the idea.

Lets recap where we are right now. We know that we have to enter 6 numbers. We know those 6 numbers must be the values 6 and below. We also know that none of those numbers can repeat. Also, note that our comparisons make use of the jbe instruction, which is for unsigned comparisons; that means the number is not negative.

Let us move on through this nightmare! We need to cut this into manageable chunks of code and try to figure it out. Sometimes there is to much code to decipher line for line and you need to figure out what is nonessential to understanding so it can be discarded. Below are chunks of code and some observations:

- Chunk 1 (0x8048e02 - 0x8048e24): You take this branch whenever ebx is >= to your input and it lands you in chunk 2. If you don't take the branch you end up in chunk 3.
- Chunk 2 (0x8048e38 - 0x8048e42): This chunk contains a loop that lands at 0x8048e10, which means all the statements above this address are coded to setup the loop. You can determine what most of the values are used for; the location ebp-0x18 is your input and edi is some sort of counter etc. The location ebp-0x34 is a mystery though.
- Chunk 3 (0x8048e26 - 0x8048e36): The odd part of this code is that the value stored at esi+0x8 gets moved into esi. So what is esi+0x8? One has to ask themselves why would a register be used as a base to overwrite itself. Currently esi holds a stack address and after the move it holds another stack address. If you examine esi's address using `x/8wx $esi` and then the address stored at esi+0x8 using `x/8wx *($esi+8)` you begin to notice a pattern. The values at esi+0x4 increment by 1 and the addresses at esi+0x8 reference how you locate the next value of your input. The first value at esi itself is some integer who's purpose is unclear but we know enough to determine these are structs that are designed to be nodes in a linked list; that satisfies our mystery from chunk 2 but opens the door on a new mystery.
- Chunk 4 (0x8048e60 - 0x8048e79): Now here is the most important chunk. We targeted it because none of the previous chunks had a call to *explode_bomb* in them; in addition we are in a better position to tackle this now because we have some idea of the variables in play. Focusing only on the comparison and the few lines before it you can surmise that the next node's first field gets compared to the current node's first field. So we solved the mystery of chunk 3. We finally know what that unknown value is, it is some number that is used to compare against other nodes. Furthermore, the bomb will explode if the current node's first field is not >= the next node's first field. So what are we doing here for the right input? We are just playing musical chairs; each node represents a chair and each chair has an associated number, the second field. What we need to do is enter the numbers 1 to 6 so that the linked list's first field numbers are in decreasing order. If they are in decreasing order they will pass the comparison everytime.

So the answer is "4 2 6 3 1 5" because that way the numbers in the linked list will be "0x3e5, 0x2d5, 0x1b0, 0x12d, 0xfd, 0xd4" the decreasing values we need.
###Secret Phase
Secrets stay secret. Try to find and solve it if you can. I won't divulge this gem because honestly this is extra credit and if you chose to do it you should not need a reference right? :-)
