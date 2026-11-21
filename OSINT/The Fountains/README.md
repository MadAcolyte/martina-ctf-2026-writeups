# The Fountains

## Challenge Information
- Event: Mārtiņa-CTF 2026
- Category: OSINT
- Author: Mārtiņš #1031

## Challenge Description

Wealth, fame, magic. Martins, the king of the pirates, attained this and everything else the world had to offer. And his dying words drove countless souls to the seas. "You want my treasure? You can have it. I left everything I gathered split below two fountains. Now you just have to find them!"

Flag format: coordinates of the XXP in the first picture followed by the coordinates of the XXP in the second picture in MGRS format, 4 digit accuracy.

Example: `MCTF26{01ABC12341234_02ABC12341234}`

## Provided Files

- Two photographs depicting different fountains.
<img width="3377" height="2652" alt="1" src="https://github.com/user-attachments/assets/fe06e641-807a-4c2e-93a3-2815fae32294" />
<img width="3430" height="3392" alt="2" src="https://github.com/user-attachments/assets/1704df92-293d-4720-abb1-88c561880283" />

## Initial Analysis

The challenge provided two photographs of fountains and required identifying the coordinates of a specific point (XXP) in each image.

I decided to use reverse image search to identify both fountains before attempting to determine their exact coordinates.

## Investigation

### 1. Identifying the Fountains

I used Google Images to perform reverse image searches on both photographs.

The search results helped me identify the locations:

<img width="658" height="418" alt="image" src="https://github.com/user-attachments/assets/39ed8b14-fbad-47e3-beb3-132dae5b6445" />

- **Fountain Nymph** – located at the Latvian National Opera House.
  
<img width="612" height="474" alt="image" src="https://github.com/user-attachments/assets/b4091e1f-e19b-42a8-82b1-967c0c22ea5f" />

- **Strūklaka Gadalaiki** – the second fountain.


### 2. Locating the Required Points

After identifying the fountains, I inspected the photographs more closely to determine the exact positions of the required XXP points.

For the first photograph, I used the visible water drainage grille as a reference point to identify the required coordinates.

<img width="1240" height="661" alt="image" src="https://github.com/user-attachments/assets/74623e31-0e3a-40c0-bcf5-3fac094b607d" />


For the second photograph, I examined the visible section of the stone wall on the left side of the image. By comparing its appearance and perspective with the actual location, I determined the angle from which the photograph had been taken and identified the required point.

<img width="1094" height="653" alt="image" src="https://github.com/user-attachments/assets/f83a3ea7-e672-4b39-ab25-e880482f4562" />


### 3. Extracting Coordinates

After identifying both required points, I obtained their geographical coordinates in decimal degrees.

- First fountain: `56.950193321576066, 24.113218256381867`
- Second fountain: `56.95166572, 24.11869665`

I then used the [LatLongData MGRS Converter](https://latlongdata.com/mgrs-converter/) to convert both coordinate pairs into MGRS format with 4-digit accuracy, as required by the challenge.

The resulting MGRS coordinates were:

- First photograph: `35VLD24431555`
- Second photograph: `35VLD24771570`

## Solution

Combining both coordinates using the required flag format resulted in:

`MCTF26{35VLD24431555_35VLD24771570}`

## Lessons Learned

- Reverse image search can help identify landmarks and narrow down possible locations.
- Small architectural details, such as drainage grilles and stone walls, can serve as useful reference points for precise geolocation.
- Identifying the exact camera perspective can be essential when locating a specific point rather than the general location of a landmark.
- MGRS provides a standardized way to represent geographical coordinates with a specified level of accuracy.
