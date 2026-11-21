# Challenge Name
The Lost Cause

## Challenge Information
- Event: Mārtiņa-CTF 2026
- Category: OSINT
- Author: Mārtiņš #404

## Challenge Description
Spatial teleportation magic mishap sent Mārtiņš to an unknown place. Maybe you can help with this? Report the location in degrees, minutes and seconds. Full seconds, no decimal seconds. 

Format reference: MCTF26{21D30M43S_173D49M1S}.

## Provided Files
- `lost-in-sauce.png` – An image depicting a section of a palace.
<img width="1024" height="512" alt="lost-in-sauce" src="https://github.com/user-attachments/assets/6d37539a-0077-41d4-83da-a258ad4f57bf" />


## Initial Analysis

The provided image showed part of a palace, but there was no immediately identifiable location or geographical information.

I decided to approach the challenge using image-based OSINT techniques.

## Investigation

### 1. Identifying the Location

I uploaded the provided image to Google Images and performed a reverse image search.

The search results helped me identify the building as **Wilanów Palace (Museum of King Jan III's Palace at Wilanów)** in Warsaw, Poland.

### 2. Verifying the Exact Location

After identifying the palace, I opened Google Maps and used Street View to compare the available views with the original challenge image.

By examining the building's architectural details and camera perspective, I found the Street View position that matched the photograph.

<img width="1275" height="655" alt="image" src="https://github.com/user-attachments/assets/9c393fa4-663e-4ca2-b892-685c20360fcd" />

### 3. Extracting Coordinates

I retrieved the coordinates of the matching camera position from Google Maps.

The coordinates were:

- Latitude: `52.165556`
- Longitude: `21.090556`

I then converted them into degrees, minutes and seconds (DMS), as required by the challenge.

- Latitude: `52°9'56"N`
- Longitude: `21°5'26"E`

## Solution

The challenge required the coordinates in the following format:

`MCTF26{latitude_longitude}`

After converting both coordinates into DMS and removing decimal seconds, the resulting flag was:

`MCTF26{52D9M56S_21D5M26S}`

## Lessons Learned

- Reverse image search can be useful for identifying unknown geographical locations.
- Google Maps Street View can help verify a location by comparing architectural details and camera perspectives.
- Geographical coordinates can be converted from decimal degrees into degrees, minutes and seconds (DMS).
