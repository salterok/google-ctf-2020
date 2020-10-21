https://capturetheflag.withgoogle.com/challenges/hardware-basics

BASICS (easy)

With all those CPU bugs I don't trust software anymore, so I came up with my custom TPM (trademark will be filed soon!). You can't break this, so don't even try.


`basics.2020.ctfcompetition.com 1337`

On connecting to above address we asked to enter password. It seems from now that provided source code represent program that handle our connection.

File `check.sv` very similar to be Verilog program (which is logical due to hardware category of this challenge).

Ok, we need `open_safe` var to become `true` which happens when `kittens == 56'd3008192072309708` (whatever that means).

Going to learn some Verilog basics to know how to control `open_safe` var.

From `main.cpp` we can see that conjunction with c `0x7f` to every char. That give us that only lower 7 bits are meaningful (ASCII range).

`always_ff` triggers on rising edge so executed once per iteration of for loop.

Variable inspection:
| register | meaning | human readable |
|--|--|--|
| `memory` | 8x7 bits  | 8 positive sign bytes |
| `idx` | 3 bit | just 3 bit [0; 7] |

On every iteration `idx` increased by 5 and current char & 0x7f is written at `idx` address of prev iteration. Taking into account that `idx` is only 3 bit wide we effectively assume 
```python
idx = (idx + 5) % 8 # on every iteration
```
Table below represent how value of `idx` change on subsequent iterations, starting to repeat at 9th step:
| 0 | 5 | 10 | 7 | 12 | 9 | 6 | 11 | 8 |
|-|-|-|-|-|-|-|-|-|
| 0 | 5 | 2 | 7 | 4 | 1 | 6 | 3 | 0 |

So Verilog number format says that `56'd3008192072309708` mean decimal 56 bit number of value `3008192072309708`.

From the definition of `magic` wire we can see that every 7 bits from `memory` mixed in specific order (0 -> 5 -> 6 -> 2 -> 4 -> 3 -> 7 -> 1) to form 56 bit number. Wire `kittens` also mixing bits around, but in differently sized groups.

So, we now try to restore `memory` value needed to produce proper `kittens`.

Value of `kittens` is `3008192072309708` in binary it looks like 
`0000101010_10111111101111010010_111110001011_01101111001100`.

Lets prepare every range used to make `kittens`:
```pseudo
magic[9:0]      ->  0000101010
magic[41:22]    ->  10111111101111010010
magic[21:10]    ->  111110001011
magic[55:42]    ->  01101111001100
```
This corresponds to value of `magic`: 
`0110111_1001100_1011111_1101111_0100101_1111000_1011000_0101010`

Now time to restore value of `memory`:
```pseudo
memory[0]   ->  0110111
memory[5]   ->  1001100
memory[6]   ->  1011111
memory[2]   ->  1101111
memory[4]   ->  0100101
memory[3]   ->  1111000
memory[7]   ->  1011000
memory[1]   ->  0101010
```

From this we can assume that value of `memory` is:
`1011000_1011111_1001100_0100101_1111000_1101111_0101010_0110111`.

Now we should remember that `memory` written in specific order for every 7 bit block: 0 -> 5 -> 2 -> 7 -> 4 -> 1 -> 6 -> 3

So lets reorder our `memory` in that order to get what we should use as input to this service:
`0110111_1001100_1101111_1011000_0100101_0101010_1011111_1111000`

For the last we represent every 7 bit block as ASCII char:
`7LoX%*_x`

And finally our flag is `CTF{W4sTh4tASan1tyCh3ck?}`.