# The Hat

## Challenge Information
- Event: Mārtiņa-CTF 2026
- Category: Misc
- Author: Mārtiņš #2**74

## Challenge Description
There is a law in the whole wizarding community. Each witch, wizard or self respecting magic practioner needs a hat. The Hat tells about the wearer more than then can... What hat will you get?

## Provided Files
- No files were provided. The challenge was solved through the CTF's Discord bot.

## Initial Analysis

The description talks about a hat that "tells about the wearer more than they can", so I assumed the hat itself would contain hidden information.

No files were attached to the challenge, so I looked at the CTF Discord server. The bot there had a slash command called `/magic-ball` with the description "Consult the Magic Ball!".

<img width="734" height="145" alt="image" src="https://github.com/user-attachments/assets/be59e11a-b305-4881-a71a-5f7c28a92946" />


I ran the command, and the bot replied with a small animated image of a wizard hat.

## Investigation

### 1. Looking at the Bot's Reply

The image the bot sent was a short animated `.webp` file that kept looping. While watching it, I noticed that text was flashing over the hat for a very short moment:

- most of the time the image showed only the hat,
- sometimes `DISCORD WARNING` flashed over it,
- for a split second, a different line of text appeared, which looked like the flag.

The `DISCORD WARNING` text was only a distraction. The flag was shown for such a short time that it was impossible to read while the animation was playing.

### 2. Capturing the Flag Frame

To read the flag, I needed to "freeze" the animation at the moment it appeared. I kept watching the loop and took a screenshot exactly when the flag text was visible.

It took a few tries to hit the right moment, but eventually I captured the frame with the flag written over the hat three times:

<img width="517" height="512" alt="Screenshot 2026-11-20 214258" src="https://github.com/user-attachments/assets/034a78b7-75a2-4d45-aa64-22d7b6021463" />

### 3. Reading the Flag

The flag was hard to read because:
- the image is very small, so the letters are only a few pixels tall,
- the text is drawn over the colourful hat and background,
- every line is cut off at the edge of the image.

Each of the three lines shows a different part of the flag, so I combined them to read as much as possible:

```
MCTF26{d1sc03d_st3g_wh0_all0w3d_ ...
       ... st3g_wh0_all0w3d_h4t_m4n ...
                    ... h4t_m4n_t?5}
```

From this I could read almost the whole flag:

`MCTF26{d1sc03d_st3g_wh0_all0w3d_h4t_m4n_t??5}`

The only part I could not read was the end. I could only see `m4n_t`, then one or two unclear characters, then `5}`.

### 4. Guessing the Ending

The rest of the flag uses leetspeak (`0` for `o`, `3` for `e`, `4` for `a`), so I guessed that the last word was a version of "this". I tried the possible endings:

- `th15`
- `th1s`
- `this`
- `thi5`

`MCTF26{d1sc03d_st3g_wh0_all0w3d_h4t_m4n_th15}` was accepted.

### 5. Solution

The final flag was:

MCTF26{d1sc03d_st3g_wh0_all0w3d_h4t_m4n_th15}

### Lessons Learned

- Not every challenge comes with a file. Sometimes the data has to be collected from somewhere else, such as a Discord bot.
- Animated images (GIF, WebP) can hide information in a frame that is only shown for a very short time.
- When text is repeated in several places, combining the readable parts can rebuild most of it.
- When only a few characters are unreadable, the flag's leetspeak pattern helps to narrow down the possible guesses.
