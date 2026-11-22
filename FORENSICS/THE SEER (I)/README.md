# The Seer (I)

## Challenge Information
- Event: Mārtiņa-CTF 2026
- Category: Forensics
- Author: Mārtiņš #1031

## Challenge Description
I have heard that you are a potent.. er... seer...

I require thine services...

I must know what the blasted Snitrams is up to...

I am sure he is scheming against me...

Find out what he is doing and inform me immediately...

## Provided Files
- `data.pcapng` – A packet capture file containing 956 USB packets.

## Initial Analysis

After opening the provided file in Wireshark, I noticed that it did not contain any network traffic. Instead, every packet was USB traffic captured with Linux [`usbmon`](https://www.kernel.org/doc/html/latest/usb/usbmon.html).

All packets came from the same device (`1.2.3`) and used an interrupt IN endpoint (`0x83`). The packets came in pairs:

- **Submit (`S`)** – 64 bytes, the host asking the device for data.
- **Complete (`C`)** – 79 bytes, the device answering with 15 bytes of data.

A single device sending short reports over an interrupt endpoint, with mostly zero bytes, is typical of a USB keyboard. Combined with the challenge description ("find out what he is doing"), my assumption was that the capture contained Snitrams' keystrokes and that I had to rebuild what he typed.

## Investigation

### 1. Extracting the HID Data

Only the `C` (URB complete) packets carry data, so I extracted their payloads with tshark:

```bash
tshark -r data.pcapng -Y 'usb.urb_type == 0x43 && usb.capdata' -T fields -e usb.capdata
```

- `-r data.pcapng` – read packets from the capture file
- `-Y '...'` – apply a [display filter](https://www.wireshark.org/docs/man-pages/wireshark-filter.html):
  - `usb.urb_type == 0x43` – keep only URB complete packets (`0x43` is the ASCII code for `C`, see the [usbmon documentation](https://www.kernel.org/doc/html/latest/usb/usbmon.html))
  - `usb.capdata` – keep only packets that carry payload data
- `-T fields -e usb.capdata` – print only the raw payload bytes

All options are described in the [tshark manual](https://www.wireshark.org/docs/man-pages/tshark.html), and the `usb.*` field names are listed in Wireshark's [USB display filter reference](https://www.wireshark.org/docs/dfref/u/usb.html).

A few of the extracted reports looked like this:

```
200201001a00000000000000000000
200201000000000000000000000000
20020100120a000000000000000000
200201021f00000000000000000000
```

### 2. Understanding the Report Format

A standard USB keyboard (boot protocol) sends 8-byte reports in the form `modifier, reserved, key1..key6`. These reports were 15 bytes, so the layout was different.

By comparing the reports, I worked out the following structure:

| Offset | Meaning |
|---|---|
| 0–2 | Constant prefix `20 02 01` (ignored) |
| 3 | Modifier byte (`0x02` = Left Shift, `0x20` = Right Shift) |
| 4–14 | Up to 11 keycodes that are currently held down |

The keycodes are HID usage IDs from the Keyboard/Keypad page of the [USB HID Usage Tables](https://usb.org/sites/default/files/hut1_5.pdf). For example:

| Keycode | Key |
|---|---|
| `0x04`–`0x1d` | `a`–`z` |
| `0x1e`–`0x27` | `1`–`0` |
| `0x28` | Enter |
| `0x2a` | Backspace |
| `0x2c` | Space |
| `0x39` | Caps Lock |

Decoding the first reports by hand gave `0x1a` = `w`, then `0x37` = `.`, which looked like the beginning of `www.`, so the assumption seemed correct.

### 3. Problems with a Simple Decoder

My first idea was to just take the first keycode from each report and map it to a character. The output was close but wrong, and I found four problems:

1. **Key rollover** – Snitrams typed fast, so a new key was often pressed before the previous one was released. For example, `12 0a` means `o` is still held and `g` was just pressed. A report lists every key that is *currently held*, so I had to compare each report with the previous one and only output newly pressed keys.
2. **Caps Lock** – The password and the flag were typed by turning Caps Lock on and off around single letters. Caps Lock has no modifier bit, so I had to remember its state myself and swap the case of letters while it was on.
3. **Shift** – Shift was used for symbols such as `{`, `}`, `@` and `%`, and for some capital letters, so I had to check both Shift bits.
4. **Backspace** – Snitrams made typos and deleted them. Printing Backspace as text would leave junk inside the flag, so I had to actually delete the last character.

### 4. Implementing a Python Decoder

To handle all of this, I wrote a decoder in Python using [Scapy](https://scapy.readthedocs.io/) to read the capture file:

```python
from scapy.utils import PcapNgReader

# HID keycode -> character (without Shift)
normal_keys = {
    0x04: "a", 0x05: "b", 0x06: "c", 0x07: "d", 0x08: "e", 0x09: "f",
    0x0a: "g", 0x0b: "h", 0x0c: "i", 0x0d: "j", 0x0e: "k", 0x0f: "l",
    0x10: "m", 0x11: "n", 0x12: "o", 0x13: "p", 0x14: "q", 0x15: "r",
    0x16: "s", 0x17: "t", 0x18: "u", 0x19: "v", 0x1a: "w", 0x1b: "x",
    0x1c: "y", 0x1d: "z",
    0x1e: "1", 0x1f: "2", 0x20: "3", 0x21: "4", 0x22: "5",
    0x23: "6", 0x24: "7", 0x25: "8", 0x26: "9", 0x27: "0",
    0x2c: " ", 0x2d: "-", 0x2f: "[", 0x30: "]", 0x33: ";",
    0x36: ",", 0x37: ".", 0x38: "/",
}

# HID keycode -> character (with Shift)
shift_keys = {
    0x1e: "!", 0x1f: "@", 0x20: "#", 0x21: "$", 0x22: "%",
    0x23: "^", 0x24: "&", 0x25: "*", 0x26: "(", 0x27: ")",
    0x2c: " ", 0x2d: "_", 0x2f: "{", 0x30: "}", 0x33: ":",
    0x36: "<", 0x37: ">", 0x38: "?",
}

ENTER = 0x28
BACKSPACE = 0x2a
CAPS_LOCK = 0x39

text = ""
caps_lock = False
previous_keys = []

for packet in PcapNgReader("data.pcapng"):
    data = bytes(packet)

    # only "C" (URB complete) packets contain keyboard data
    if len(data) <= 64 or data[8] != ord("C"):
        continue

    report = data[64:]  # skip the 64-byte usbmon header
    shift = report[3] == 0x02 or report[3] == 0x20  # left or right Shift
    keys = report[4:]

    for key in keys:
        # skip empty slots and keys that are still held from the last report
        if key == 0 or key in previous_keys:
            continue

        if key == CAPS_LOCK:
            caps_lock = not caps_lock
        elif key == BACKSPACE:
            text = text[:-1]
        elif key == ENTER:
            text += "\n"
        else:
            if shift and key in shift_keys:
                char = shift_keys[key]
            else:
                char = normal_keys.get(key, "?")
                if shift:
                    char = char.upper()

            if caps_lock and char.isalpha():
                char = char.swapcase()

            text += char

    previous_keys = list(keys)

print(text)
```

The script works as follows:
1. Reads every packet from `data.pcapng`.
2. Keeps only URB complete (`C`) packets, because only they contain keyboard data.
3. Skips the 64-byte usbmon header to get the HID report.
4. Checks the modifier byte to see if Left or Right Shift is held.
5. Skips empty key slots and keys that were already held in the previous report.
6. Toggles Caps Lock, deletes the last character on Backspace, and adds a new line on Enter.
7. Converts every other keycode to a character, applying Shift and Caps Lock.
8. Prints the rebuilt text.

### 5. Solution

I saved the implementation as `solve.py` and executed it in Kali Linux:

```bash
python3 solve.py
```

<img width="344" height="322" alt="image" src="https://github.com/user-attachments/assets/fabea01e-553c-4617-8da6-777e7a0438f8" />


The decoded keystrokes were:

```
www.google.com
how do i get flags
free flags
flag online free
snitrams.melnais@gmail.com
S3cur3P455w0Rd
MCTF26{m4RT1N5_kn0w5_Wh4t_y0u_wr0T3}
youtube.com
funny verity clips
verity backrooms or minecraft
```

So this is what Snitrams was up to: searching for free flags, logging into his account `snitrams.melnais@gmail.com` with the password `S3cur3P455w0Rd`, and then typing the flag.

Looking at the raw keystrokes for the flag line shows why the Caps Lock and Backspace handling was needed:

```
<CAPS>MCTF26<CAPS>{m4RT1n<BS><CAPS>N<CAPS>%<BS>5_kn0w5_<CAPS>W<CAPS>h4t_y0u_wr0T3}
```

Without handling Backspace, the result would be `m4RT1nN%5_...`, and without handling Caps Lock it would start with `mctf26{`.

The final flag was:

MCTF26{m4RT1N5_kn0w5_Wh4t_y0u_wr0T3}

### Lessons Learned

- A packet capture does not always contain network traffic. USB traffic can also be captured and analysed in Wireshark.
- USB keyboards send their keystrokes as HID reports, which makes it possible to rebuild everything a user typed.
- Keyboard reports describe which keys are currently held down, not individual key presses, so each report has to be compared with the previous one.
- Not every keyboard uses the standard 8-byte boot report, so the report format should be checked before writing a decoder.
- Caps Lock, Shift and Backspace all have to be handled, otherwise the decoded flag will be wrong.
- Captured USB keyboard traffic can expose sensitive information such as email addresses and passwords, which is why malicious USB devices and hardware keyloggers are a real threat.

## References
- [tshark manual page](https://www.wireshark.org/docs/man-pages/tshark.html)
- [Wireshark display filter syntax](https://www.wireshark.org/docs/man-pages/wireshark-filter.html)
- [Wireshark USB display filter fields](https://www.wireshark.org/docs/dfref/u/usb.html)
- [Linux usbmon documentation](https://www.kernel.org/doc/html/latest/usb/usbmon.html)
- [USB HID Usage Tables 1.5](https://usb.org/sites/default/files/hut1_5.pdf)
- [Scapy documentation](https://scapy.readthedocs.io/)
