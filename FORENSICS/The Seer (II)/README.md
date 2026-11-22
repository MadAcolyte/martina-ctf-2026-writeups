# The Seer (II)

## Challenge Information
- Event: Mārtiņa-CTF 2026
- Category: Forensics
- Author: Mārtiņš #1031

## Challenge Description
It is... not enough...

I... must know more...

I MUST know more...

Seer into his head...

Find out what he is doing...

## Provided Files
- `data.pcapng` – A packet capture file containing 10,994 USB packets.

## Initial Analysis

Like in [The Seer (I)](../The%20Seer%20(I)/README.md), the capture contained no network traffic, only USB traffic captured with Linux [`usbmon`](https://www.kernel.org/doc/html/latest/usb/usbmon.html).

Almost all packets again came from device `1.2.3` over the interrupt IN endpoint `0x83`, in Submit (`S`, 64 bytes) and Complete (`C`, 79 bytes) pairs. The only other packets were 3 short control transfers on endpoint `0x00`, which did not contain anything useful.

<img width="1277" height="739" alt="image" src="https://github.com/user-attachments/assets/645c5af2-1be1-4220-80cf-eca72caea9d8" />

This time there were 5,494 reports instead of a few hundred, and the challenge asked me to "seer into his head". My first guess was that Snitrams was no longer typing but doing something else with the same device.

## Investigation

### 1. Extracting the HID Data

I extracted the payloads of the `C` packets with the same tshark command as in part I:

```bash
tshark -r data.pcapng -Y 'usb.urb_type == 0x43 && usb.capdata' -T fields -e usb.capdata
```

- `-r data.pcapng` – read packets from the capture file
- `-Y '...'` – apply a [display filter](https://www.wireshark.org/docs/man-pages/wireshark-filter.html):
  - `usb.urb_type == 0x43` – keep only URB complete packets (`0x43` is the ASCII code for `C`, see the [usbmon documentation](https://www.kernel.org/doc/html/latest/usb/usbmon.html))
  - `usb.capdata` – keep only packets that carry payload data
- `-T fields -e usb.capdata` – print only the raw payload bytes

All options are described in the [tshark manual](https://www.wireshark.org/docs/man-pages/tshark.html), and the `usb.*` field names are listed in Wireshark's [USB display filter reference](https://www.wireshark.org/docs/dfref/u/usb.html).

<img width="1437" height="778" alt="image" src="https://github.com/user-attachments/assets/cc9b4ea7-ef03-4b52-a105-948f9f109d33" />


A few of the extracted reports looked like this:

```
200102000001f0ff00000000000000
2001020000ff1f0000000000000000
200102010000e0ff00000000000000
200201010600000000000000000000
```

### 2. Understanding the Report Format

Almost all reports (5,488 of 5,494) started with `20 01 02` instead of the `20 02 01` keyboard prefix from part I. The bytes after it also did not look like keycodes: byte 3 was only ever `0x00` or `0x01`, and bytes 5–7 were full of values like `ff`, `f0` and `e0`, which are typical of small negative numbers.

This is what a USB mouse report looks like: a button byte followed by relative X/Y movement. The receiver seems to be a combo keyboard + mouse device, which uses the second and third bytes to tell the two kinds of reports apart.

By comparing the reports, I worked out the following structure for the mouse reports:

| Offset | Meaning |
|---|---|
| 0–2 | Prefix `20 01 02` (mouse report) |
| 3 | Buttons (`0x01` = left button held) |
| 4 | Always `0x00` (ignored) |
| 5–7 | X and Y movement, packed as two signed 12-bit numbers |
| 8–14 | Always zero (ignored) |

The movement is packed into 3 bytes (24 bits = 2 × 12 bits):

```
X = byte5 | (low 4 bits of byte6) << 8
Y = (high 4 bits of byte6) | byte7 << 4
```

Both values are signed ([two's complement](https://en.wikipedia.org/wiki/Two%27s_complement)), so anything at or above `0x800` is negative. For example, for `200102010000e0ff...`:

- buttons = `0x01` → left button held
- bytes 5–7 = `00 e0 ff`
- X = `0x000` = 0
- Y = `0xffe` = −2 → the mouse moved 2 units up while the button was pressed

The movement values are relative, so the absolute cursor position is found by adding up all the movements from the start.

### 3. The Keyboard Reports

The remaining 6 reports used the keyboard prefix `20 02 01` from part I:

```
200201010600000000000000000000
```

<img width="1568" height="315" alt="image" src="https://github.com/user-attachments/assets/9fbb1596-2273-4621-9956-ab0bed176dd8" />


Using the same layout as before, the modifier `0x01` is Left Ctrl and the keycode `0x06` is `c` (see the [USB HID Usage Tables](https://usb.org/sites/default/files/hut1_5.pdf)). So the only keys pressed in the whole capture were **Ctrl+C**, twice, at the very end, after all the mouse movement. This was probably just Snitrams closing the program, so it was not needed for the flag.

### 4. Implementing a Python Decoder

Since Snitrams was holding the left button and moving the mouse, he was most likely drawing something. To see it, I wrote a Python script that rebuilds the cursor path with [Scapy](https://scapy.readthedocs.io/) and draws every line made while the left button was held with [Matplotlib](https://matplotlib.org/stable/api/pyplot_summary.html):

```python
from scapy.utils import PcapNgReader
import matplotlib.pyplot as plt


def to_signed_12bit(value):
    # 12-bit two's complement -> Python int
    return value - 0x1000 if value & 0x800 else value


x, y = 0, 0
strokes = []        # list of finished lines
current_stroke = [] # points of the line being drawn right now

for packet in PcapNgReader("data.pcapng"):
    data = bytes(packet)

    # only "C" (URB complete) packets contain HID data
    if len(data) <= 64 or data[8] != ord("C"):
        continue

    report = data[64:]  # skip the 64-byte usbmon header

    # keep only mouse reports (20 01 02 ...)
    if report[:3] != b"\x20\x01\x02":
        continue

    buttons = report[3]
    dx = to_signed_12bit(report[5] | ((report[6] & 0x0f) << 8))
    dy = to_signed_12bit((report[6] >> 4) | (report[7] << 4))

    # movements are relative, so add them up to get the cursor position
    x += dx
    y += dy

    if buttons & 0x01:  # left button held -> drawing
        current_stroke.append((x, y))
    elif current_stroke:  # button released -> line finished
        strokes.append(current_stroke)
        current_stroke = []

if current_stroke:
    strokes.append(current_stroke)

fig, ax = plt.subplots(figsize=(10, 7))
for stroke in strokes:
    xs, ys = zip(*stroke)
    ax.plot(xs, ys, "k-", linewidth=2)

ax.invert_yaxis()  # screen Y grows downwards
ax.set_aspect("equal")
ax.axis("off")
plt.savefig("drawing.png", bbox_inches="tight")
print(f"Drew {len(strokes)} strokes, saved to drawing.png")
```

The script works as follows:
1. Reads every packet from `data.pcapng`.
2. Keeps only URB complete (`C`) packets, because only they contain HID data.
3. Skips the 64-byte usbmon header to get the HID report.
4. Keeps only mouse reports (prefix `20 01 02`) and ignores the Ctrl+C keyboard reports.
5. Unpacks the two signed 12-bit X/Y values and adds them to the current cursor position.
6. Records the cursor position while the left button is held, and starts a new line every time the button is released.
7. Draws all lines and flips the Y axis, because screen coordinates grow downwards.
8. Saves the result as `drawing.png`.

### 5. Solution

I saved the implementation as `solve.py` and executed it in Kali Linux:

```bash
python3 solve.py
```

<img width="326" height="125" alt="image" src="https://github.com/user-attachments/assets/7fcca7bd-cafe-4c7d-b577-423872ca423a" />


The script found 50 separate strokes and produced this image:


<img width="590" height="378" alt="image" src="https://github.com/user-attachments/assets/49d2e16e-377c-4901-9931-8f86e5cf25c4" />


Snitrams had drawn the flag by hand, followed by two small faces. Reading it line by line:

```
MCTF26{
m4rt1nn1A
_s33r_4g3
NCy}
```

The final flag was:

MCTF26{m4rt1nn1A_s33r_4g3NCy}

### Lessons Learned

- The same USB receiver can send reports for several devices (here a keyboard and a mouse), so the reports have to be separated before decoding them.
- USB mice report relative movement, not absolute positions, so the cursor path has to be rebuilt by adding up every movement.
- Mouse reports do not always use whole bytes for X and Y. Here they were packed as 12-bit signed values, so bit operations and two's complement were needed to decode them.
- Plotting the cursor path only while a button is held turns captured mouse traffic back into whatever the user drew.
- Not only keystrokes can be recovered from USB captures. Mouse movement can also reveal what a user was doing, including handwritten text, signatures or drawn patterns.

## References
- [tshark manual page](https://www.wireshark.org/docs/man-pages/tshark.html)
- [Wireshark display filter syntax](https://www.wireshark.org/docs/man-pages/wireshark-filter.html)
- [Wireshark USB display filter fields](https://www.wireshark.org/docs/dfref/u/usb.html)
- [Linux usbmon documentation](https://www.kernel.org/doc/html/latest/usb/usbmon.html)
- [USB HID Usage Tables 1.5](https://usb.org/sites/default/files/hut1_5.pdf)
- [Two's complement](https://en.wikipedia.org/wiki/Two%27s_complement)
- [Scapy documentation](https://scapy.readthedocs.io/)
- [Matplotlib pyplot documentation](https://matplotlib.org/stable/api/pyplot_summary.html)
