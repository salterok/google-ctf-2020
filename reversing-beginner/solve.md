https://capturetheflag.withgoogle.com/challenges/reversing-beginner

BEGINNER (easy)

Dust off the cobwebs, let's reverse!

In attachment we have `a.out` file, looks like Linux executable.
>\> file ./a.out    
> ./a.out: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, 
BuildID[sha1]=e3a5d8dc3eee0e960c602b9b2207150c91dc9dff, for GNU/Linux 3.2.0, not stripped

Try to execute:
>\> ./a.out  
> Flag: 1   
> FAILURE   

Searching text like 'flag' and 'ctf' reveals that they appears in file, but no raw flag present.    
>\> string ./a.out  
strncmp     
puts        
printf      
Flag:       
SUCCESS     
FAILURE     
CTF{    
`

Looks like we need to analyze program flow to reverse the flag.

Lets dump ASM to some file for later analysis
>\> objdump -d ./a.out > asm.dump.txt

We're happy that binary is not stripped as showed earlier by `file` util. So we can remove all unnecessary code block and focus on `main` function.

After we enter any value that program asks, we observe that in placed on xmm0 register and then sequentially executes `pshufb`, `paddd` and `pxor` ops.
Max 15 chars read from input, so that with NULL-byte ending it ends up at 16 bytes (128bit xmm operand).

Those operations executed with next mask operands:  
>.data:0000555555558070 SHUFFLE xmmword 0D0C0A08040F030E090B0501070602h 
>.data:0000555555558060 ADD32 xmmword 6763746613371337FEE1DEADDEADBEEFh 
>.data:0000555555558050 XOR xmmword 0AAF986EB34F823D4385F1A8D49B45876h  

Lets reverse asm to some pseudo code so we can more clearly see how input is checked:
```c++
EXPECTED_PREFIX = "CTF{"
scanf("%15s", input);
check = 
    xor(
        add(
            shuffle(input, SHUFFLE),
            ADD32
        ),
        XOR
    );
if (
    !strncmp(input, check, 16) &&
    strncmp(check, EXPECTED_PREFIX, 4) == 0 
)
{
    puts("SUCCESS");
}
else
{
    puts("FAILURE");
}
```

From above pseudo code we can conclude that program prints `SUCCESS` only when `input` equals to `check` and starts with `CTF{`.    

So our task for now is to find cases when aforementioned computations on `input` produce the same value.    

Also we could see that `input` is not initialized to any value before `scanf`, so in case we enter less then 15 chars, there will be some garbage. That provide additional restriction on input data – whole flag must be 15 char long.

Above is a table that displays all known data, where `INP` – original input, `RES` – expected output (should be same as `INP`), `SHF` – mask to `pshufb`, `ADD` - operand to `paddd`, `XOR` – operand for `pxor`, `X`*n* – designates unknown value.  
> NOTE: last byte should be NULL due to how user input handled, also the byte before last is assumed to be `7D` (`}`) to form proper flag.

```
INP: 43 54 46 7B X4 X5 X6 X7 X8 X9 XA XB XC XD 7D 00
SHF: 02 06 07 01 05 0B 09 0E 03 0F 04 08 0A 0C 0D 00
ADD: EF BE AD DE AD DE E1 FE 37 13 37 13 66 74 63 67
XOR: 76 58 B4 49 8D 1A 5F 38 D4 23 F8 34 EB 86 F9 AA
RES: 43 54 46 7B X4 X5 X6 X7 X8 X9 XA XB XC XD 7D 00
```

Lets write down a table to see how `pshufb` shuffle bytes around:

| Original index    | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | A | B | C | D | E | F |
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
| Result index      | F | 3 | 0 | 8 | A | 4 | 1 | 2 | B | 6 | C | 5 | D | E | 7 | 9 |

I spend several hours pushing the wrong way due to assumption that `paddd` work on every byte separately, but it doesn't.
You can see the way i think [here](fail_path.md).

Lets compute where every byte ended after `pshufb` and call that value `SRE`:
```
INP: 43 54 46 7B X4 X5 X6 X7  X8 X9 XA XB XC XD 7D 00   

SRE: 46 X6 X7 54 X5 XB X9 7D  7B 00 X4 X8 XA XC XD 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B X4 X5 X6 X7  X8 X9 XA XB XC XD 7D 00   
```
The only idea till now is to move from least significant bytes and compute every `X`*n* possible.
> Moving from least significant bytes should be safe because they not affected by `ADD` carry value or we already know that value.  

Short check if table do not contradict to itself.    
Use very last column values: 43 + 67 = AA ^ 00, which is true.

`XD` + `63` = `F9` ^ `7D` = `84`    
`XD` = `21` => !    

Now substitute `XD` with its value and repeat.
> `PRO` row is to save my sanity and clearly see whats left unknown.    
```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ?? ?? ?? ?? ?? ?? ++ ++   
SRE: 46 X6 X7 54 X5 XB X9 7D  7B 00 X4 X8 XA XC 21 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B X4 X5 X6 X7  X8 X9 XA XB XC 21 7D 00   
```
`XC` + `74` = `86` ^ `21` = `A7`    
`XC` = `33` => 3    
```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ?? ?? ?? ?? ?? ++ ++ ++   
SRE: 46 X6 X7 54 X5 XB X9 7D  7B 00 X4 X8 XA 33 21 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B X4 X5 X6 X7  X8 X9 XA XB 33 21 7D 00   
```
`XA` + `66` = `EB` ^ `33` = `D8`    
`XA` = `72` => r    
```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ?? ?? ?? ?? ++ ++ ++ ++   
SRE: 46 X6 X7 54 X5 XB X9 7D  7B 00 X4 X8 72 33 21 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B X4 X5 X6 X7  X8 X9 72 XB 33 21 7D 00   
```
Now it's interesting situation we have `X8` and `XB` unknown and can't compute this column and all next to the left.    
Ok, in fact we can skip this column and safely compute next one if we can prove that no carry can happen. But this is not the case, so we also can compute `X4`.    
What if we generalize previous assumption in that way: *we can compute some column if we can prove that no carry comes from previous columns*.    

`X9` is another unknown we can't compute right now, but `X8` looks more promising. `SRE` and `ADD` value of previous column is `00` and `13` respectively, carry can't grow that much from previous columns the it will overflow to next column.    
So now we safely compute `X8`:  
`7B` + `37` = `D4` ^ `X8` = `B2`  
`X8` = `66` => f    
```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ++ ?? ?? ?? ++ ++ ++ ++   
SRE: 46 X6 X7 54 X5 XB X9 7D  7B 00 X4 66 72 33 21 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B X4 X5 X6 X7  66 X9 72 XB 33 21 7D 00   
```
By solving `X8` we made possible to solve `XB`: 
`66` + `13` = `34` ^ `XB` = `79`    
`XB` = `4D` => M    
```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ++ ?? ?? ++ ++ ++ ++ ++   
SRE: 46 X6 X7 54 X5 4D X9 7D  7B 00 X4 66 72 33 21 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B X4 X5 X6 X7  66 X9 72 4D 33 21 7D 00   
```
It's clear now that no overflow happen from previous column, so we can resolve `X4` now:    
`X4` + `37` = `F8` ^ `72` = `8A`    
`X4` = `53` => S    
```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ++ ?? ++ ++ ++ ++ ++ ++   
SRE: 46 X6 X7 54 X5 4D X9 7D  7B 00 53 66 72 33 21 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 X5 X6 X7  66 X9 72 4D 33 21 7D 00   
```
`00` + `13` = `23` ^ `X9` = `13`    
`X9` = `30` => 0    
```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 X6 X7 54 X5 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 X5 X6 X7  66 30 72 4D 33 21 7D 00   
```
We fully compute last half of the token, first half only has 3 unknowns left — `X5`, `X6` and `X7`. 

And again i made a mistake by moving in wrong direction `right-to-left` when actually less significant bytes to the left side.

You can review my false calculations in [fail #2](fail_path.md).

> ! try to interpret bytes from left as least significant
> second part of table has no overflow so we do not need to recalculate values
> but first half has many overflows and order of bytes very important
> as overflow caries to higher bytes

First i want to check that right part of table (solved one) do not contradicts to itself.
This time i will use some simple script to do this, cause my mind will blow otherwise :)

I just copied right part and with simple text replacements made below js array
```js
[
    [ 0x7B, 0x00, 0x53, 0x66, 0x72, 0x33, 0x21, 0x43 ],
    [ 0x37, 0x13, 0x37, 0x13, 0x66, 0x74, 0x63, 0x67 ],
    [ 0xD4, 0x23, 0xF8, 0x34, 0xEB, 0x86, 0xF9, 0xAA ],
    [ 0x66, 0x30, 0x72, 0x4D, 0x33, 0x21, 0x7D, 0x00 ],
]
```
Now execute following script to see that all rows draws `true`
```js
for (let i = 0; i < 8; i++) {
    let afterSum = a[0][i] + a[1][i];
    let afterXor = a[2][i] ^ a[3][i];
    let ok = ((afterSum - afterXor) & 0xFF) === 0;
    console.log(`${i+1}: ${ok}`);
}
```
And they indeed in sync! Ok, so we now need to carefully look into solving left part of table `left-to-right` this time.

---

```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 X6 X7 54 X5 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 X5 X6 X7  66 30 72 4D 33 21 7D 00   
```

`46` + `EF` = `76` ^ `43` = `35` (`135`)
and 1 carried to next byte

```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 X6 X7 54 X5 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BF AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 X5 X6 X7  66 30 72 4D 33 21 7D 00   
```

`X6` + `BF` = `58` ^ `54` = `0C`
`X6` = `4D` => M
> `X6` = `0C` - `BF` < 0, that means overflow happened and significant part of value carried to next byte and current one left with small value of `0C`
```
PRO: ++ ++ ?? ++ ?? ?? ?? ??  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 4D X7 54 X5 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BF AE DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 X5 4D X7  66 30 72 4D 33 21 7D 00   
```
`X7` + `AE` = `B4` ^ `46` = `F2`
`X7` = `F2` - `AE` = `44` => D  *no overflow this time*

```
PRO: ++ ++ ++ ++ ?? ?? ?? ??  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 4D 44 54 X5 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BF AE DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 X5 4D 44  66 30 72 4D 33 21 7D 00   
```
> ! `54` + `DE` = `49` ^ `7B` = `32`

`X5` + `AD` = `8D` ^ `53` = `DE`
`X5` = `DE` - `AD` = `31` => 1
```
PRO: ++ ++ ++ ++ ++ ?? ?? ??  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 4D 44 54 31 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BF AE DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 31 4D 44  66 30 72 4D 33 21 7D 00   
```

`4D` + `DE` = `1A` ^ `31`
`2B` (`12B`) = `2B`

```
PRO: ++ ++ ++ ++ ++ ++ ?? ??  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 4D 44 54 31 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BF AE DE AD DE E2 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 31 4D 44  66 30 72 4D 33 21 7D 00   
```

`30` + `E2` = `5F` ^ `4D`
`12` (`112`) = `12`

```
PRO: ++ ++ ++ ++ ++ ++ ++ ??  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 4D 44 54 31 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BF AE DE AD DE E2 FF  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 31 4D 44  66 30 72 4D 33 21 7D 00   
```

`7D` + `FF` = `38` ^ `44`
`7C` (`17C`) = `7C`
> This particular carry goes nowhere as this is top byte in `QWORD`

```
PRO: ++ ++ ++ ++ ++ ++ ++ ++  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 4D 44 54 31 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BF AE DE AD DE E2 FF  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 31 4D 44  66 30 72 4D 33 21 7D 00   
```

Check left side of table with simple script used earlier
```js
[
    [ 0x46, 0x4D, 0x44, 0x54, 0x31, 0x4D, 0x30, 0x7D ],
    [ 0xEF, 0xBF, 0xAE, 0xDE, 0xAD, 0xDE, 0xE2, 0xFF ],
    [ 0x76, 0x58, 0xB4, 0x49, 0x8D, 0x1A, 0x5F, 0x38 ],
    [ 0x43, 0x54, 0x46, 0x7B, 0x53, 0x31, 0x4D, 0x44 ],
]
```

```js
for (let i = 0; i < 8; i++) {
    let afterSum = a[0][i] + a[1][i];
    let afterXor = a[2][i] ^ a[3][i];
    let ok = ((afterSum - afterXor) & 0xFF) === 0;
    console.log(`${i+1}: ${ok}`);
}
```

All is ok, now time to compose whole flag and feed it to our program.

Flag is: `CTF{S1MDf0rM3!}`

>\> ./a.out     
>Flag: CTF{S1MDf0rM3!}      
>SUCCESS

