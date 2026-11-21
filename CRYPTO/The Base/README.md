# The Base

## Challenge Information
- Event: Mārtiņa-CTF 2026
- Category: Cryptography
- Author: Mārtiņš #1031

## Challenge Description
All Your Base Are Belong To Us.

## Provided Files
- `flag.txt` – A text file containing multiple lines of numbers, each represented using a different numerical base.
  
Content:
```
11111111111111111111111111111111111111111111111111111111111111111111111111111
1000011
10010
1012
200
130
234
166
56
82
100
7b
77
3a
38
33
40
55
62
2c
2b
4j
43
4n
22
21
2o
44
38
1j
3n
3t

```

## Initial Analysis

After opening the provided file, I noticed that it contained multiple lines of numbers represented using different numerical bases.

My initial assumption was that each line was encoded using a different base, starting with Base1, followed by Base2, Base3, and so on.

I decided to test this assumption by manually decoding the first few lines using [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Base(4)&input=MTAxMg).

## Investigation

### 1. Identifying the Encoding Pattern

I started by examining the first lines of the file.

The first line consisted entirely of `1` characters, which suggested Unary encoding (Base1).

I then tested the following lines using different numerical bases:

| Value | Base | Decimal | ASCII |
|---|---:|---:|---|
| `1000011` | 2 | 67 | C |
| `10010` | 3 | 84 | T |
| `1012` | 4 | 70 | F |

The resulting characters formed the beginning of the expected flag prefix:

`CTF`

This confirmed my assumption that each subsequent line used an incrementing base.

### 2. Implementing a Python Decoder

After identifying the pattern, I decided to automate the decoding process using Python instead of manually decoding every line.

Python provides the built-in `int(value, base)` function, which converts a string representing a number in a given base into its decimal integer value.

However, Python does not support Base1 through this function. Therefore, I implemented a separate condition for the first line.

For Unary encoding, each character represents one unit, so the decoded value is simply the length of the string.

My implementation was:

```python
def decode(value, base):
    if base == 1:
        return len(value)

    return int(value, base)


base = 1

with open("flag.txt", "r") as file:
    for line in file:
        value = line.strip()
        print(chr(decode(value, base)), end="")
        base += 1
```
The script works as follows:
1. Starts with Base1.
2. Reads the file line by line.
3. Removes unnecessary whitespace using strip().
4. Decodes the current line using its corresponding base.
5. Converts the resulting decimal value into a character using chr().
6. Increments the base for the next line.

### 3. Solution
I saved the implementation as decoder.py and executed it in Kali Linux:

<img width="370" height="94" alt="image" src="https://github.com/user-attachments/assets/04e56278-6660-4c21-a1f5-ffd928bf390f" />

The final decoded output was:

MCTF26{v3Ry_b453D_t45k_w45Nt_1t}

### Lessons Learned

- Different numerical bases can represent the same integer using different digit systems.
- Python's built-in int(value, base) function makes decoding positional numeral systems straightforward.
- Unary (Base1) encoding requires separate handling because it is not supported by Python's int() function.
- Python's chr() function converts decoded integer values into their corresponding Unicode characters.
- Recognizing patterns in encoded data can significantly reduce the time needed to solve CTF challenges.
- Automating repetitive decoding operations is more efficient and less error-prone than manually decoding every line.
