---
title: Computers Cannot Multiply
date: 2024-09-18
tags:
  - software-horrors
---

To a first approximation, computers cannot multiply. What would you call a function that is incorrect 99.999999% of the time? Broken?

Let’s take a look: if you multiply two (unsigned) 4-bit numbers

|      | 0000 | 0001 | 0010 | 0011 | 0100 | 0101 | 0110 | 0111 | 1000 | 1001 | 1010 | 1011 | 1100 | 1101 | 1110 | 1111 | 
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 | 0000 |
| 0001 | 0000 | 0001 | 0010 | 0011 | 0100 | 0101 | 0110 | 0111 | 1000 | 1001 | 1010 | 1011 | 1100 | 1101 | 1110 | 1111 |
| 0010 | 0000 | 0010 | 0100 | 0110 | 1000 | 1010 | 1100 | 1110 | NO | NO | NO | NO | NO | NO | NO | NO |
| 0011 | 0000 | 0011 | 0110 | 1001 | 1100 | 1111 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 0100 | 0000 | 0100 | 1000 | 1100 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 0101 | 0000 | 0101 | 1010 | 1111 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 0110 | 0000 | 0110 | 1100 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 0111 | 0000 | 0111 | 1110 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 1000 | 0000 | 1000 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 1001 | 0000 | 1001 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 1010 | 0000 | 1010 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 1011 | 0000 | 1011 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 1100 | 0000 | 1100 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 1101 | 0000 | 1101 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 1110 | 0000 | 1110 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |
| 1111 | 0000 | 1111 | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO | NO |

- All numbers multiplied by zero or one are okay.
- If we multiply by 2, half the numbers are incorrect.
- If we multiply by 3, two-thirds of the numbers are incorrect.
- If we multiply by 4, three-quarters of the numbers are incorrect.
- ...

So for four-bit numbers, out of 256 possible combinations, 180 results are incorrect, and 76 results are correct. This makes the result incorrect about 70% of the time.

But: it gets much, much worse the more bits your numbers have. The possible value space increases dramatically, but the same results as above apply: when multiplying by two, half are incorrect, etc, etc. This means that when we use longer numbers, there are more incorrect results:

| Length | Possible results     |    Incorrect results | Percentage incorrect |
| -----: | -------------------: | -------------------: | -------------------: |
|      8 | 65536                |                63568 | 97%                  |
|     16 | 4294967296           |           4294099268 | 99.98%               |
|     32 | 18446744073709552000 | 18446743969190916110 | 99.999999%           |
Thus, 32-bit multiplication is incorrect 99.999999% of the time.