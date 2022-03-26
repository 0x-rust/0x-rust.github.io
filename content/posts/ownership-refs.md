---
title: "Values references, Ownership "
date: 2022-03-09T19:07:58Z
draft: true
---

## Value and Ownership
All values in rust have a single owner (this is enforced  by the compiler), what this means is one scope is responsible for deallocating the value.
Rust single ownership invariant means in every scope there is a single variable which owns
a value.
In the example below, single ownership invariant is broken, as we have two owners x, y to the same value.   

```
struct Pair(String, i64);
//x owns  the value 
let x = Pair("100".to_string(), 100);

//x reliquises ownership  
let y = x;

let same = x.1 == 100; //illegal as x no longer owns the value
```
when x is assinged to y, Rust **moves** the value to y. 
For most types this is default semantic when assignment happens



