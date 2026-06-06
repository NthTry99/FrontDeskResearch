<!-- markdownlint-disable MD033 -->
<!-- markdownlint-disable MD041 -->
<link rel="stylesheet" href="../style.css">

<h1>IBM ThinkPad T40 EEPROM CRC1, CRC2, and SVP CRC Calculators</h1>
<h2>Fixing BIOS POST CRC Errors</h2>

<b>NthTry99</b><br>February 2026

<h2>Table of Contents</h2>

- [1. Project Summary](#1-project-summary)
  - [1.1 Quick Background Info](#11-quick-background-info)
  - [1.2 What You Will Need](#12-what-you-will-need)
- [2. What is CRC1, CRC2, and the SVP CRC?](#2-what-is-crc1-crc2-and-the-svp-crc)
  - [2.1 CRC1/CRC2 Calculator](#21-crc1crc2-calculator)
  - [2.2 SVP CRC Calculator](#22-svp-crc-calculator)
  - [2.3 Checksums, not CRCs](#23-checksums-not-crcs)
- [3. Reverse Engineering the CRCs and Lost Knowledge](#3-reverse-engineering-the-crcs-and-lost-knowledge)
  - [3.1 Alleged Location and Calculation](#31-alleged-location-and-calculation)
  - [3.2 How the CRC Checks Work](#32-how-the-crc-checks-work)
  - [3.3 The Hunt Begins](#33-the-hunt-begins)
  - [3.4 CRC1 and Block 0 Oddities](#34-crc1-and-block-0-oddities)
  - [3.5 Sourcing EEPROM Data](#35-sourcing-eeprom-data)
  - [3.6 Block 0 Analysis](#36-block-0-analysis)
  - [3.7 Taking a Break: The SVP CRC](#37-taking-a-break-the-svp-crc)
  - [3.8 CRC1 and CRC2 Decoded](#38-crc1-and-crc2-decoded)
  - [3.9 Why Did IBM Do it This Way?](#39-why-did-ibm-do-it-this-way)
  - [3.10 Final Testing Info](#310-final-testing-info)

---

## <a id="1-project-summary"></a>1. Project Summary

This project focuses on IBM and early Lenovo-era ThinkPads (such as an IBM ThinkPad T40) and the various POST errors which are caused by problems with data stored in the EEPROM. Typically they occur due to data corruption as a result of the EEPROM chip failing or a power failure during a data write to the EEPROM. Regardless, these EEPROM-related POST errors can effectively brick the laptop. Back in the day (and possibly still) the official solution was a complete system board replacement (aka a new motherboard).

In this writeup, I will be focusing on the following POST error codes(1):

- 0175: Bad CRC1. The EEPROM checksum is not correct. Usually attributed to data corruption. IBM recommends replacing motherboard.
- 0177: Bad SVP data. Checksum of the supervisor password in the EEPROM is not correct. IBM recommends replacing motherboard.
- 0182: Bad CRC2. Enter BIOS Setup and load Setup defaults. Checksum of the CRS2(2) setting in the EEPROM is not correct.

(1) = DIRECTLY FROM THE OFFICIAL IBM T40 HARDWARE MAINTENANCE MANUAL (page 54): [http://ps-2.kev009.com/pccbbs/mobiles_pdf/92p1286.pdf](http://ps-2.kev009.com/pccbbs/mobiles_pdf/92p1286.pdf) or [https://archive.org/details/ibm-service-manual-thinkpad-t40.pdf](https://archive.org/details/ibm-service-manual-thinkpad-t40.pdf)
<br>
(2) = [this is likely a typo in the official documentation and should say CRC2]

In theory, if these CRCs could be located in the EEPROM and the method of their calculation was known, an external programmer could be used to manually write the expected CRC directly to the correct memory offsets (if the chip is still functional) or physically replace the EEPROM chip and flash a complete, valid (and compatible) dump to it. You could then enter the laptop's serial number, UUID, etc. as well as the expected CRC to the EEPROM and it would fix the laptop without having to replace the entire motherboard.

I got into this project as a result of my previous one where I extracted the Supervisor Password (SVP) from a ThinkPad T40 by dumping the data from the EEPROM chip via an external programmer and decoding it from there. I already have a bunch of EEPROM data from that project and I wanted to see what else I could learn from it. From a little research and reading the IBM T40 Hardware Maintenance Manual, I found out that these errors are also directly related to the data on the EEPROM chip, which I am already familiar with extracting and modifying.

---

***Disclaimer: This project is my best attempt to reverse engineer the calculation process of CRC1, CRC2, and the SVP CRC within the EEPROM of an IBM ThinkPad T40. I only have one of these laptops to obtain genuine EEPROM data from. Although I was able to find a handful of EEPROM dumps from similar-era ThinkPads online, I can only assume that this EEPROM data is authentic. Due to these factors, I cannot be 100% certain that my findings are completely accurate, and there is no official documentation from IBM regarding this very specific, somewhat sensitive (albeit legacy) technical information. However, I do have a fair amount of data and successful results to believe that my findings (and IBM CRC Calculator and SVP CRC Calculator scripts) have a decent probability of being correct.***

The only way I could be more certain is if I obtained an IBM ThinkPad T40 (or similar-era ThinkPad) that actually has each these CRC errors (0175, 0177, and 0182), dumped the EEPROM data from it, verified that the CRC in the data is different than what my program calculates, then replace it with the correctly calculated CRC, write the changes back to the EEPROM chip, and then verify that the laptop functions properly. I could try to recreate these errors and risk permanently ruining my T40, or I guess I will have to keep an eye out for these ancient laptops with these very specific errors on eBay... There could be an update one day!

Another possibility: I have figured out how to generate/calculate the specific bytes which I THINK are the CRCs but they are actually not the CRCs. Nevertheless I can very consistently and accurately calculate these bytes.

***As always, if you do not understand something, ASK! Ask me or look it up. This writeup is purely for informational purposes. I am not responsible for any damage caused by using the information in this document.***

### <a id="11-quick-background-info"></a>1.1 Quick Background Info

If you haven't read my previous project writeup about extracting the Supervisor Password (SVP) from the EEPROM, here is some quick background info that is pertinent to this project:

<b>How Data is Stored in the EEPROM:</b>
<br>
The IBM ThinkPad T40 uses an Atmel AT24RF08CN EEPROM chip. From an I2C perspective, this EEPROM is organized into 4 256-byte main data blocks (addressed 0x54-0x57), and an additional 32-byte block (addressed 0x5c) which stores both a 16-byte Access Protection Page (APP) and a 16-byte ID information page (ID page).

The main memory blocks are what store the laptop's/component's serial numbers, UUID, Supervisor Password (SVP) and other configuration data/flags.

The APP stores data which, depending on their values, can prevent reading/writing protected parts of the EEPROM (such as the block where the SVP is stored).

The ID page stores "an ID value used for asset identification" according to the datasheet. However in reality I am not sure exactly what this data is. I only have one set of this ID page data from my T40, which doesn't change under any circumstance (making any kind of differential analysis impossible), nor does it seem to be relevant to anything I have done with this laptop.

For the context of this guide, just know that each CRC is calculated using specific data within one of the main data blocks (and stored there too), the SVP is a specific 7-byte set of data, and the APP/ID is an additional 32-byte set of data. According to very limited online resources, the SVP data and APP/ID data are sometimes used as a mask/salt during the CRC calculation.

### <a id="12-what-you-will-need"></a>1.2 What You Will Need

- HxD or similar hex editor (if you want to view the EEPROM data)
- Python (to run my scripts)
- general computer/PowerShell/terminal knowledge
- general understanding of computer memory
- general understanding of checksums/CRCs

## <a id="2-what-is-crc1-crc2-and-the-svp-crc"></a>2. What is CRC1, CRC2, and the SVP CRC?

I will begin with the TL;DR SparkNotes version of my findings, and follow it with the details of how I came to my conclusion and wrote my IBM CRC Calculator programs.

In short, after much trial and error and differential analysis, my working theory is as follows:

- CRC1 is at 0x08
- CRC2 is at 0x208
- SVP CRC is at 0x33F and repeated at 0x347

### <a id="21-crc1crc2-calculator"></a>2.1 CRC1/CRC2 Calculator

To use my CRC1/CRC2 calculator, open a terminal or Command Prompt/PowerShell in a directory containing "IBM_CRC_CALCULATOR_V2.py" as well as whatever full 1024-byte EEPROM dump whose CRC1/CRC2 you want to calculate. Then execute the following command:

Windows:
<br>
<small><i>powershell:</i></small>

```powershell
python .\IBM_CRC_CALCULATOR_V2.py yourDump.bin
```

---OR---

<br>
Linux:
<br>
<small><i>bash:</i></small>
<br>

```bash
python3 IBM_CRC_CALCULATOR_V2.py yourDump.bin
```

This script should work for any IBM ThinkPad and some early Lenovo ThinkPad models. I will give more details in the long-form section below, but I have successfully tested it on 6 different EEPROM dumps from laptops ranging from 2001 through 2013.

### <a id="22-svp-crc-calculator"></a>2.2 SVP CRC Calculator

To use my SVP CRC Calculator, open a terminal or Command Prompt/PowerShell in a directory containing "svp_checksum_V2.py" as well as whatever full 1024-byte EEPROM dump whose SVP CRC you want to calculate. Then execute the following command:

Windows:
<br>
<small><i>powershell:</i></small>

```powershell
python .\svp_checksum_V2.py yourDUMP.bin
```

---OR---

<br>
Linux:
<br>
<small><i>bash:</i></small>
<br>

```bash
python3 svp_checksum_V2.py yourDUMP.bin
```

### <a id="23-checksums-not-crcs"></a>2.3 Checksums, not CRCs

Technically, none of these are CRCs (Cyclic Redundancy Checks):

- CRC1 and CRC2 are checksum "balancer bytes" (aka complement bytes) which are specifically calculated and stored in their respective locations for the purpose of making the sum of all data in the memory block a multiple of 0x100 (decimal 256).
- The SVP CRC is even simpler: it is a checksum8 modulo 256 stored at the byte immediately following the 7 bytes representing the SVP scancodes (beginning at 0x338 and repeated at 0x340). In other words, the sum of the bytes modulo 256.
- For the sake of consistency, convenience and legacy however, I will continue to refer to them as CRCs.

Back in this era (the T40 was released in 2003), the terms CRC (Cyclic Redundancy Check) and checksum were frequently used interchangeably in technical documentation, even though they represent different mathematical operations. So it is quite plausible that my theory is correct.

The remaining sections of this report will be the long-form narrative of my findings and how I wrote my CRC calculators.

## <a id="3-reverse-engineering-the-crcs-and-lost-knowledge"></a>3. Reverse Engineering the CRCs and Lost Knowledge

There were 2 primary questions to answer:

1. Where in the EEPROM are these CRCs stored?
2. How exactly are these CRCs calculated?

Neither of these problems were made easier by the fact that there is close to no pertinent information anywhere online. To say this specific topic is a niche is an understatement. Most of my findings were direct results of my own analysis. There were also some vague and questionable suggestions which came from online repair forums (and some Google Gemini), which may have helped point me in the right direction, but at face value were often incorrect (such as the CRC locations or how they were calculated).

To further complicate matters, I was unable to document/save a lot of the sources I did find. During the entirety of this project, I was battling with my 16-year-old computer constantly crashing due to failing hardware and therefore losing documentation and information pulled up in incognito browser tabs (which is a bad habit I am trying to break). Luckily, at the present time, the internet and AI is still available and halfway useful, so much of the uncited information can be readily found.

### <a id="31-alleged-location-and-calculation"></a>3.1 Alleged Location and Calculation

<b>Location:</b>
<br>
Originally, documentation and repair forums for old IBM ThinkPads said CRC1 and CRC2 were located at 0x2E-0x2F and 0x22E-0x22F, respectively. Or that they were the very last 2 bytes of data in each block. Or the very last 2 bytes of the entire memory block.

<b>Calculation:</b>
<br>
Repair forums suggested that these CRCs are calculated using a specific 16-bit CRC algorithm: CRC-16-CCITT using polynomial 0x1021. And they use a specific seed value, usually 0x0000, 0xFFFF or rarely a different seed value. Also, each CRC calculation could potentially use the SVP bytes as a mask (i.e the raw data XORed against the password hex data) and/or use the 32-byte APP/ID data as a salt. In summary, the process was allegedly this:

<div style="margin-left: 2em;">
Raw Data (omitting the CRC byte(s)) --> XOR with Supervisor Password --> XOR with APP/ID Salt --> CRC Calculation (using a fixed seed) --> Result (stored at CRC location)
</div>
<br>

The logic behind this process is:

- The APP/ID salt binds the data to the motherboard.
- The SVP mask binds the data to the user (the user created the password).
- The seed and polynomial provide the mathematical proof that both the motherboard and user are correct.

### <a id="32-how-the-crc-checks-work"></a>3.2 How the CRC Checks Work

When the laptop changes data on the EEPROM chip within these CRC-protected areas, it calculates a CRC of the protected data and then stores it in the EEPROM memory. Then, when the computer POSTs (aka boots) it calculates the same CRC of the same data on the fly and compares it to the stored CRC. If the freshly calculated CRC does not match the expected (stored) CRC, you get one of the aforementioned errors, and your computer has likely transformed into a brick.

### <a id="33-the-hunt-begins"></a>3.3 The Hunt Begins

At this point I had the supposed location and calculation info, so I set out to calculate the CRCs and find them in the data. I would start with CRC1. I began by writing a Python script to calculate the CRC (a 2-byte value) using the CRC-16-CCITT algorithm, polynomial 0x1021, variable seed values, and the SVP mask and APP/ID salt from the full dump. Then I tested every combination of these variables in search of the expected CRCs.

Not only were there many different variables at this point, I also really wasn't even sure what result (the CRC) I was looking for. I had my sights on the two possible locations mentioned earlier, and made sure for each test calculation that my script omitted these CRC bytes during the calculation (the CRC can't include itself). I never got any of the expected CRC values at either location. Or any location. Or at the end of the data. Or end of the block.

Next I tried a different seed value. In the first tests I tried 0x0000, so I changed it to the other likely seed: 0xFFFF. Still, I didn't get my calculated CRC1 to match up with any data at the potential locations. At this point I wondered if maybe a custom seed was being used. So I wrote a modified version of my script to brute-force determine what seed (if any) would result in the values at the various locations. Well, mathematically, of course there is a seed that will generate any desired output value, but I still wouldn't know which one (if any) was correct unless I 100% knew the correct target CRC value. I decided to dig deeper into the location of these CRCs.

### <a id="34-crc1-and-block-0-oddities"></a>3.4 CRC1 and Block 0 Oddities

I combed through block 0's data and quickly realized I had been looking in the wrong place. I have pasted the exact data dump for block 0 below (note that this computer is 23 years old now and is purely a research item to me, it really doesn't matter if any of its info/specs are publicly available):

<small><i>block 0 hex data:</i></small>

```text
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f    0123456789abcdef
00: 53 45 52 23 14 0c 01 d2 e1 00 00 00 00 00 00 00    SER#?????.......
10: 40 31 33 52 31 31 32 33 5a 4a 31 55 52 59 34 33    @13R1123ZJ1URY43
20: 50 31 52 47 20 b1 53 32 33 37 33 37 32 55 39 39    P1RG ?S237372U99
30: 35 5a 47 4e 36 00 00 00 08 33 38 4c 35 30 30 31    5ZGN6...?38L5001
40: 5a 4a 31 4e 55 50 34 34 31 30 35 44 00 00 00 00    ZJ1NUP44105D....
50: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
60: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
70: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
90: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
```

Some of the ASCII looked very familiar. Between this project and my original SVP decoding project, I had looked at this data literally hundreds of times already but somehow never put together the fact that "SER#" and the information that follows it might be the Serial Number and other hardware info... revolutionary I know.

This was a huge breakthrough. All the locations I previously thought were the CRC are actually fragments of Vital Product Data (VPD, which is basically hardware configuration information). For example, 0x2E-0x2F are digits in the system-unit serial number. After this realization, I set out to determine exactly what all of this data represents; if I could correctly correlate the data to verifiable configuration info, the remaining unknown data would be where I would start to look for CRC1.

Based on the fact that all of the supposed locations for CRC1 were in block 0, I assumed this was CRC1's block. Also, block 1 (which starts with the ASCII-translated "USR#") has absolutely no data in it, other than the header, so I do not believe it has its own CRC. Block 2 is the CON# block and does have data stored in it, including sensitive data like the UUID, and most info I found about CRC2 says it is in this block, but I will talk more about this later. Finally, block 3 is the SVP block, and the SVP has its own "CRC" which I will also talk about later.

### <a id="35-sourcing-eeprom-data"></a>3.5 Sourcing EEPROM Data

At this point, I went online and found as many EEPROM dumps as I could from similar ThinkPads from the same era as my T40. As an interesting side note, to find these, I simply Google searched "53 45 52 23" which are the first 4 values in the EEPROM dump (remember, this translates to "SER#" and every dump of this era (and likely more) would begin with these values). I had a pretty good feeling I would find at least a few dumps this way since when users have/had problems with their EEPROM dumps, they would simply copy-paste their entire dump onto a forum and ask the community for help.

So from here onward, when I talk about all the dumps I used for differential analysis, I am referring to the following laptop EEPROM dumps: my T40 (often named some variation of 0x54-0x57.bin) a T22.BIN, R51.BIN, T400.BIN, and T61 (I renamed it T61russia.BIN because it was from a semi-legitimate Russian website, and I had another partial T61.BIN I wanted to differentiate it from).

I also have one from a much later Lenovo-era ThinkPad, L540.BIN, which I will occasionally reference.

### <a id="36-block-0-analysis"></a>3.6 Block 0 Analysis

After a lot of research and looking through the BIOS menu for serial numbers, UUID, and whatever other unique data I could find, here is my preliminary belief as to what the data represents:

<b>Offsets 0x00-0x03:</b>
<br>
53 45 52 23 (SER#) is just a label for the block.

<b>Offsets 0x04-0x07:</b>
<br>
14 0C 01 D2 (...Ò) Not completely sure what these are, but I am pretty confident they are not the CRC because from the 5 different IBM ThinkPad dumps I compared, they are all the same values. Each of these dumps were from different laptops with unique product data (i.e serial number etc) and therefore the CRC would be almost certainly have to be different for each. Only in the 2013 Lenovo L540.BIN did these bytes change, which is consistent with them being some sort of version/revision data.

<b>Offset 0x08*:</b>
<br>
E1: THIS is what I believe to be CRC1. In IBM's architecture for this era, they sometimes used Header-Resident CRCs, placing the CRC at the end of the header.
<br>
\* = Originally, I believed it was both 0x07-0x08, since the result from the CRC-16-CCITT algorithm/polynomial is a 2-byte value.

I am pretty confident the rest of the data is accounted for and does not include the CRC. Nevertheless, the remaining data are as follows:

<b>Offsets 0x10-0x18:</b>
<br>
40 31 33 52 31 31 32 33 5A (@13R1123Z) is the IBM FRU (Field Replaceable Unit) Number or the Part Number for the blank motherboard itself before it was assigned a serial number (@ is a delimiter).
<br>
13R1123 is a specific laptop processor socket part number for this era of IBM ThinkPads. (you can Google this and find evidence supporting this). The Z likely signifies it is ROHS compliant (Restriction of Hazardous Substances).

<b>Offsets 0x19-0x23:</b>
<br>
4A 31 55 52 59 34 33 50 31 52 47 (J1URY43P1RG) This is the system board serial number. Verified within the BIOS menu and sticker on bottom of laptop.

<b>Offsets 0x24-0x26:</b>
<br>
20 B1 53 ( ±S) I believe to be a delimiter because it looks like one and also these offsets are the exact same in my block 0 dump as well as the other dumps I compared.

<b>Offsets 0x27-0x34:</b>
<br>
32 33 37 33 37 32 55 39 39 35 5A 47 4E 36 (237372U995ZGN6) This is the type and system-unit serial number (visible in the BIOS).

From the sticker on the bottom of the laptop:
<div style="margin-left: 2em;">
Type: 2373-72U
<br>
S/N 99-5ZGN6
</div>
<br>

<b>Offsets 0x35-0x37:</b>
<br>
Blank data (all 00)

<b>Offset 0x38:</b>
<br>
08 (.) I believe to be another delimiter

<b>Offsets 0x39-0x3F:</b>
<br>
33 38 4C 35 30 30 31 (38L5001) This is the IBM part number (FRU) of the CPU (you can Google this and our 1.50GHz Pentium M pops up)

<b>Offsets 0x40-0x4B</b>
<br>
5A 4A 31 4E 55 50 34 34 31 30 35 44 (ZJ1NUP44105D) This is most likely the 12-digit individual serial number of this CPU. This is the only data that changes from specific laptop model EEPROM dump to dump I compared that I cannot 100% account for. However, since in all cases the hex does translate into ASCII characters and immediately follows known VPD data, I believe it is just more of that data and not CRC related.

Additionally, through differential analysis of multiple dumps of my T40 (i.e. changing SVP and settings between dumps) I noticed the SER# (block 0) never changes. Which makes perfect sense because all of this product data is independent of any settings and should never change.

So with all of that determined, I was pretty confident that if block 0 is indeed the data used to calculate CRC1, that 0x07-0x08 was its location (at this point I was assuming that 0x07 was part of the CRC since the CRC-16-CCITT result should be a 2-byte value).

I also suspected that CRC2 was in the same relative location in block 2. Not only did the repair forums say block 2 held CRC2, but block 1 was completely empty except for a 10-byte header including the ASCII-translated text "USR#" and block 3 is the SVP block, which has its own verification.

Although there is no official IBM information available that maps the specific hex offsets for these CRCs, my location guess is plausible. It is consistent with a header-resident validation structure sometimes used in IBM's hardware configuration blocks, where for some memory blocks they would have an identification header (ex. "SER#") followed by a CRC for the block.

With my attention now focused on the headers of blocks 0 and 2, I also determined that for block 2, 0x204 was the length of the data and 0x206 was likely a version/revision number (in all dumps 0x206 was either 01 or 02, older models seemed to be 01 which supports this). So all data within the header was accounted for except the last 2 bytes, which (at the time) further validated my belief they were the CRC.

I once again tried brute-force calculating the seed needed to get my target CRC at 0x07-0x08. With every combination of including or omitting the APP/ID salt and the SVP mask, I was able to find seeds that would generate CRC1. I also did this process trying to get CRC2 (at 0x207-0x208) in block 2. However, when I tried using those same parameters (seed, mask, salt etc) on other EEPROM dumps, it would never yield the actual CRC in that data, which means it was a "howler" solution (the right answer for the wrong reason), not the actual CRC calculation process.

### <a id="37-taking-a-break-the-svp-crc"></a>3.7 Taking a Break: The SVP CRC

I switched gears and decided to go after the SVP CRC. After all, the SVP is at most 7 characters, how hard could it be? Luckily it was a much more straightforward process, and ultimately led to the solution of CRC1 and CRC2.

From information on repair forums, here is how the SVP CRC works: during POST verification, if an SVP is set, the BIOS calculates the sum of the stored password (the scancodes). If this sum does not match the trailing validation byte, the system triggers the 0177 error.

I quickly confirmed the SVP CRC is a checksum8 modulo 256 stored at the byte immediately following the 7 bytes representing the SVP scancodes. I wrote a script (called svp_checksum_V2.py) to calculate the sum of the SVP hex data and compare it to the stored SVP checksum.

To further verify this, I performed several iterations of changing the SVP via the BIOS menu and dumping that new EEPROM data after each password change. After 7 or so new sets of this data, I can confidently say this trailing byte is a checksum of the 7-byte SVP data.

Here is some example data of this SVP checksum calculation:

<b>From 0x54-0x57COPY.bin. Password is MUDGE21</b>
<br>
<small><i>In hex data beginning at 0x338:</i></small>

```text
32 16 20 22 12 03 02 A1
M  U  D  G  E  2  1
```

CheckSum8 Modulo 256:
<br>
Sum of Bytes % 256  = Checksum Byte
<br>
0x32 + 0x16 + 0x20 + 0x22 + 0x12 + 0x03 + 0x02 = A1
<br>
A1 is the checksum, so it is omitted in the calculation. The sum of the first 7 bytes is 161 (in decimal). 161 % 256 = 161, and 161 in hex is A1

<br>

<b>From R51.bin from internet. Password is 12345</b>
<br>
<small><i>In hex data:</i></small>

```text
02 03 04 05 06 00 00 14
1  2  3  4  5 
```

CheckSum8 Modulo 256:
<br>
Sum of Bytes % 256  = 14

<br>

<b>From abcdefg.bin from internet. Password is ABCDEFG</b>
<br>
<small><i>In hex data:</i></small>

```text
1E 30 2E 20 12 21 22 F1
A  B  C  D  E  F  G 
```

CheckSum8 Modulo 256:
<br>
Sum of Bytes % 256  = F1

<br>

<b>From 0x54-0x57heyalex.bin. Password is HEYALEX</b>
<br>
<small><i>In hex data:</i></small>

```text
23 12 15 1E 26 12 2D CD
H  E  Y  A  L  E  X
```

CheckSum8 Modulo 256:
<br>
Sum of Bytes % 256  = CD

### <a id="38-crc1-and-crc2-decoded"></a>3.8 CRC1 and CRC2 Decoded

With this new knowledge, and on a hunch, I wondered "what if CRC1 and CRC2 are just checksums too?" After all, in the official IBM Hardware and Maintenance Manual, it does call them checksums. I had been assuming they were CRCs largely because that's what online resources say they are, and that's what they are called in the POST errors.

I used a hex addition script I made earlier to find the sums of blocks 0 and 2. While the result in both cases were not the CRCs I was looking for, I did notice very quickly that for all 5 similar-era ThinkPad dumps (and my multiple T40 dumps), the CRC1 and CRC2 blocks always added up to multiples of 0x100.

After some more research about memory blocks adding up to nice round numbers (as well as other data validation methods of the era) I learned about this type of classic checksum validation (specifically a two's complement checksum) and the use of checksum bytes/complement bytes. I changed my CRC calculator program to calculate both "CRCs" using this new theory.

So for the SER# (block 0) CRC1 is the complement byte/balancer byte at 0x08. It only has to be whatever value is needed for the sum of the block to add up to a multiple of 0x100.

For the CON# (block 2) CRC2 is the complement byte/balancer byte at 0x208. Interestingly, there seems to be a relationship between CRC2, the byte before it (at 0x207), the block-length byte at 0x204, and the supposed revision number at 0x206.

From my own T40 and changing settings/SVPs, at first I thought both 0x207-0x208 were checksum bytes. I actually figured out that these two bytes can BOTH be calculated based off of the rest of the block data by using the block-length byte (0x204), the supposed revision number (at 0x206) and a common constant (dependent on the revision number) to accurately divide up the addends (amongst 0x207-0x208) needed to achieve the block-total of a multiple of 0x100.

In retrospect, I believe it is possible that the special circumstances between CRC2, the length-byte, version, and previous byte are irrelevant; 0x207 may just be some other internally used value or flag of some sort and CRC2 is simply an additive checksum, not related to the length-byte or version byte. Nevertheless, my CRC1/CRC2 calculator script will accurately calculate both values.

### <a id="39-why-did-ibm-do-it-this-way"></a>3.9 Why Did IBM Do it This Way?

Perhaps IBM (or later Lenovo) used an algorithm/polynomial for calculating a true CRC in EEPROMs for later generations of computers/laptops. But for the T40 (and probably many other legacy systems) it seems they just used a simple additive checksum verification system. This is likely because this style of data integrity verification is fast and simple; to check integrity, the system simply adds the entire block (including the balancer) and if the result isn't the expected multiple, the data is considered corrupt.

### <a id="310-final-testing-info"></a>3.10 Final Testing Info

My CRC calculator accurately calculates the expected "CRC1" (0x08) and "CRC2" (0x208), as well as the preceding byte, for every ThinkPad EEPROM dump I have in this directory (all from circa 2003) WITH THE EXCEPTION of R51.BIN, for which CRC2 ("CON#", block 2) is inaccurate. However the sum of that block is also not a multiple of 0x100. I believe I obtained this R51.BIN from a forum post dated to the early-mid 2000's, when the laptop was likely still in use. If that is the case, then it is likely that the uploader sanitized the data (i.e modified his UUID, etc) prior to uploading, therefore affecting the data/sum/checksum.

I also tried with a 2013 ThinkPad L540 EEPROM dump, which my calculator does work for, though there are noticeable differences in the data in this model vs older IBM units. Since I only had one of these later dumps to work with, I only have this one point of reference data and am therefore not as confident with the dependability of my program with these later ThinkPad "CRC" calculations.

For both CRC1 and CRC2, I further verified its functionality by zeroing out CRC1 (and preceding byte) at 0x07-0x08 and CRC2 (and preceding byte) at 0x207-0x208 and my script will correctly provide the missing values.

In addition to my CRC calculator scripts aforementioned, I have included a few various "debug" or test scripts in a directory named "research_tools". One of which is a hex data sum calculator, and another is a script used to brute-force find the seed(s) which produce the expected output "CRCs". Since these were purely for my research use, they should not be considered finished products (or even relevant) and may need to be run from within an IDE and possibly require some code tweaking.

<script>
  document.title = "IBM ThinkPad EEPROM CRC Calculators";
</script>
