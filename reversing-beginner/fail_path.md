```
INP: 43 54 46 7B X4 X5 X6 X7 X8 X9 XA XB XC XD XE 7D
SHF: 02 06 07 01 05 0B 09 0E 03 0F 04 08 0A 0C 0D 00
ADD: EF BE AD DE AD DE E1 FE 37 13 37 13 66 74 63 67
XOR: 76 58 B4 49 8D 1A 5F 38 D4 23 F8 34 EB 86 F9 AA
RES: 43 54 46 7B X4 X5 X6 X7 X8 X9 XA XB XC XD XE 7D
```

| Original index    | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | A | B | C | D | E | F |
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
| Result index      | F | 3 | 0 | 8 | A | 4 | 1 | 2 | B | 6 | C | 5 | D | E | 7 | 9 |

Now we want to write down formula for every `X`*n* which will describe how `INP` and `RES` connected. 
For example lets look at first byte `X1` (we know it's value is `43`).  
XOR 43 76 must yield the same value as ADD 46 EF, lets check:
```
0x43 ^ 0x76 => 53
(0x46 + 0xEF) & 0xFF => 53
```
> Note: `ADD` drop overflow so result of addition is effectively passed though MODULE `0xFF + 1`

Generally we can write down formula like:
> `X`*n* ^ `XOR`[*n*] = `INP`[`SHF`[*n*]] + `ADD`[*n*]

Now time to write formulas for every unknown `X`*n*, *n* ∈ {`4`,`5`,`6`,`7`,`8`,`9`,`A`,`B`,`C`,`D`,`E`}:

> X4 ^ 8D = X5 + AD     
> X5 ^ 1A = XB + DE     
> X6 ^ 5F = X9 + E1     
> X7 ^ 38 = XE + FE     
> X8 ^ D4 = 7B + 37     
> X9 ^ 23 = 7D + 13     
> XA ^ F8 = X4 + 37     
> XB ^ 34 = X8 + 13     
> XC ^ EB = XA + 66     
> XD ^ 86 = XC + 74     
> XE ^ F9 = XD + 63    

----
```
INP: 43 54 46 7B X4 X5 X6 X7 X8 X9 XA XB XC XD XE 7D
SHF: 02 06 07 01 05 0B 09 0E 03 0F 04 08 0A 0C 0D 00
ADD: EF BE AD DE AD DE E1 FE 37 13 37 13 66 74 63 67
XOR: 76 58 B4 49 8D 1A 5F 38 D4 23 F8 34 EB 86 F9 AA
RES: 43 54 46 7B X4 X5 X6 X7 X8 X9 XA XB XC XD XE 7D
```
| Original index    | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | A | B | C | D | E | F |
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
| Result index      | F | 3 | 0 | 8 | A | 4 | 1 | 2 | B | 6 | C | 5 | D | E | 7 | 9 |
----

We have 2 constants `X8` and `X9`, lets compute them:
> X8 ^ D4 = 7B + 37 = B2    
> X8 = 66

> X9 ^ 23 = 7D + 13 = 90       
> X9 = B3

Now substitute `X8` and `X9` with computed values to be able to compute another unknowns:

> X4 ^ 8D = X5 + AD     
> X5 ^ 1A = XB + DE     
> X6 ^ 5F = B3 + E1     
> X7 ^ 38 = XE + FE         
> XA ^ F8 = X4 + 37     
> XB ^ 34 = 66 + 13     
> XC ^ EB = XA + 66     
> XD ^ 86 = XC + 74     
> XE ^ F9 = XD + 63    

Continue to compute variables one-by-one and substitute them to resolve every char:     

> X6 ^ 5F = B3 + E1 = 94    
> X6 = CB

> XB ^ 34 = 66 + 13 =        
> XB = 


Oh, assumed `paddd` behavior is wrong, need to rethink 
In real `paddd` add every 8 bytes as whole so we can't restore every byte that easy.

--------------

