```
PRO: ++ ?? ?? ++ ?? ?? ?? ??  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46 X6 X7 54 X5 4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF BE AD DE AD DE E1 FE  37 13 37 13 66 74 63 67   
XOR: 76 58 B4 49 8D 1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43 54 46 7B 53 X5 X6 X7  66 30 72 4D 33 21 7D 00   
```

`7D` + `FE` = `38` ^ `X7` = (`17B` & `FF`) = `7B`   
`X7` = `43` => C    
> NOTE: first overflow happens so we carry 1 to next column, also we mark every next column with `?` next to `ADD` value so that we know that such value may possible got also overflowed (this is just to self check and remove any uncertainty why constant expressions don't match). 
```
PRO: ++  ??  ??  ++  ??  ??  ?? ++  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46  X6  43  54  X5  4D  30 7D  7B 00 53 66 72 33 21 43   
ADD: EF? BE? AD? DE? AD? DE? E2 FE  37 13 37 13 66 74 63 67   
XOR: 76  58  B4  49  8D  1A  5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43  54  46  7B  53  X5  X6 43  66 30 72 4D 33 21 7D 00   
```
`30` + `E2` = `5F` ^ `X6` = (`112` & `FF`) = `12`   
`X6` = `4D` => M    
Substitute `X6` in table in correct values in `ADD` due to overflow:    
```
PRO: ++  ??  ??  ++  ??  ?? ++ ++  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46  4D  43  54  X5  4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF? BE? AD? DE? AD? DF E2 FE  37 13 37 13 66 74 63 67   
XOR: 76  58  B4  49  8D  1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43  54  46  7B  53  X5 4D 43  66 30 72 4D 33 21 7D 00   
```
`4D` + `DF` = `1A` ^ `X5` = (`12C` & `FF`) = `2C`   
`X5` = `36` => 6    
```
PRO: ++  ??  ??  ++  ??  ++ ++ ++  ++ ++ ++ ++ ++ ++ ++ ++   
SRE: 46  4D  43  54  36  4D 30 7D  7B 00 53 66 72 33 21 43   
ADD: EF? BE? AD? DE? AE  DF E2 FE  37 13 37 13 66 74 63 67   
XOR: 76  58  B4  49  8D  1A 5F 38  D4 23 F8 34 EB 86 F9 AA   
RES: 43  54  46  7B  53  36 4D 43  66 30 72 4D 33 21 7D 00   
```

> YET AGAIN SOMETHING GOT WRONG
36 + AE != 8D ^ 53

