---
title: Computers Cannot Multiply
date: 2024-09-18
permalink: computers-cannot-multiply
tags:
  - software-horrors
---

What would you call a function that is incorrect 99.999999% of the time? Broken?

Let’s take a look: what happens if you multiply two (unsigned) 4-bit numbers?

<table  style="font-family: monospace !important; text-align: center">
    <thead>
        <tr>
            <th>×</th>
            <th>0000</th>
            <th>0001</th>
            <th>0010</th>
            <th>0011</th>
            <th>0100</th>
            <th>0101</th>
            <th>0110</th>
            <th>0111</th>
            <th>1000</th>
            <th>1001</th>
            <th>1010</th>
            <th>1011</th>
            <th>1100</th>
            <th>1101</th>
            <th>1110</th>
            <th>1111</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <th scope="row">0000</th>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
            <td>0000</td>
        </tr>
        <tr>
            <th scope="row">0001</th>
            <td>0000</td>
            <td>0001</td>
            <td>0010</td>
            <td>0011</td>
            <td>0100</td>
            <td>0101</td>
            <td>0110</td>
            <td>0111</td>
            <td>1000</td>
            <td>1001</td>
            <td>1010</td>
            <td>1011</td>
            <td>1100</td>
            <td>1101</td>
            <td>1110</td>
            <td>1111</td>
        </tr>
        <tr>
            <th scope="row">0010</th>
            <td>0000</td>
            <td>0010</td>
            <td>0100</td>
            <td>0110</td>
            <td>1000</td>
            <td>1010</td>
            <td>1100</td>
            <td>1110</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">0011</th>
            <td>0000</td>
            <td>0011</td>
            <td>0110</td>
            <td>1001</td>
            <td>1100</td>
            <td>1111</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">0100</th>
            <td>0000</td>
            <td>0100</td>
            <td>1000</td>
            <td>1100</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">0101</th>
            <td>0000</td>
            <td>0101</td>
            <td>1010</td>
            <td>1111</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">0110</th>
            <td>0000</td>
            <td>0110</td>
            <td>1100</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">0111</th>
            <td>0000</td>
            <td>0111</td>
            <td>1110</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">1000</th>
            <td>0000</td>
            <td>1000</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">1001</th>
            <td>0000</td>
            <td>1001</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">1010</th>
            <td>0000</td>
            <td>1010</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">1011</th>
            <td>0000</td>
            <td>1011</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">1100</th>
            <td>0000</td>
            <td>1100</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">1101</th>
            <td>0000</td>
            <td>1101</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">1110</th>
            <td>0000</td>
            <td>1110</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
        <tr>
            <th scope="row">1111</th>
            <td>0000</td>
            <td>1111</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
            <td>NO</td>
        </tr>
    </tbody>
</table>

- All numbers multiplied by zero or one are okay.
- If we multiply by 2, half the numbers are incorrect.
- If we multiply by 3, two-thirds of the numbers are incorrect.
- If we multiply by 4, three-quarters of the numbers are incorrect.
- etc

So for four-bit numbers, out of 256 possible combinations, 180 results are incorrect, and 76 results are correct. This makes the result incorrect about 70% of the time.

But: it gets much, much worse the more bits your numbers have. The possible value space increases dramatically, but the same results as above apply: when multiplying by two, half are incorrect, etc, etc. This means that when we use longer numbers, there are more incorrect results:

| Length | Possible results     |    Incorrect results | Percentage incorrect |
| -----: | -------------------: | -------------------: | -------------------: |
| 4 | 256 | 180 | 70% |
|      8 | 65536                |                63568 | 97%                  |
|     16 | 4294967296           |           4294099268 | 99.98%               |
|     32 | 18446744073709552000 | 18446743969190916110 | 99.999999%           |

Thus, 32-bit multiplication is incorrect 99.999999% of the time.

The next fun thing to consider is why computers seem to be able to multiply, despite not being able to.
