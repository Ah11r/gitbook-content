# Forensics Day 1

I finally made the decision, I am delving deeper into forensics. I want to be the forensics swiss-army-knife 🤓\
I will utilise TryHackMe in learning the basics. Ideally below are the rooms I managed to cover and what I learnt from each.

### [Intro to Digital Forensics](https://tryhackme.com/room/introdigitalforensics)

This is a very basic room suitable for every beginner. It introduced me to the realm of digital forensics where I learned the following;

* Digital forensics is the process of collecting and analyzing digital evidence
* Being a process, it follows some defined steps/stages which are;\
  a. Evidence collection\
  b. Creating a chain of custody\
  c. Securing the evidence\
  d. Transportation of the evidence\
  e. Creating a copy of the evidence\
  f. Processing and analyzing the evidence\
  g. Reporting

In this room, I also learnt how to perform a basic forensics on pdf and image files. The tools for the respective files are pdfinfo and exiftool.\
To use pdfinfo tool, one has to install it using the below command

```
sudo apt update && sudo apt install poppler-utils
```

The syntax for analyzing the pdf metadata is: _**pdfinfo file.pdf**_\
Some of the metadata we can acquire using the tool pdfinfo include: author, creation time, encoded messages and more.

Exif tool follows the same syntax and can be used to analyze both pdf files and image files. i.e _**exiftool image.png | exiftool file.pdf**_\
This tool can also be used to hide/encode some info within an image.

### [DFIR: An Introduction](https://tryhackme.com/room/introductoryroomdfirmodule)

This was also an interesting room building upon the previous room. Here, I learnt some more detailed concepts as below.

1. There exists a strong correlation between digital forensics and incident response. i.e. Incident respose relies on digital forensics while digital forensics gathers its goal and scope from incident response (DFIR)
2. Artifacts are pieces of evidence that point to an activity performed on a system
3. Evidence preservation is key. Thus the first step after gaining collecting the evidence is write protecting it and taking a copy
4. A secure chain of custody should be maintained to preserve the integrity of the evidence collected
5. Order of volatility should be considered when performing digital forensics. The more volatile sources of evidence should be collected first.
6. Inorder to make the evidence fully understandable, a meaningful timeline of events should be created in a chronological manner.

This room introduced me to the various tools used for digital forensics process. The good thing is that TryHackme has a room dedicated to each of the tools which makes it easier to gain familiarity. Below are the mentioned tools which I look forward to learning in the coming days

1. Eric Zimmerman's tools - to learn these tools, I will complete [Windows Forensics 1](https://tryhackme.com/room/windowsforensics1) and [Windows Forensics 2](https://tryhackme.com/room/windowsforensics2)
2. [KAPE](https://tryhackme.com/room/kape) - `Kroll Artifact Parser and Extractor - used to automate collection and parsing of artifacts`
3. [Autopsy](https://tryhackme.com/room/btautopsye0) - `open source forensics platform that helps analyze data from digital media`
4. [Volatility](https://tryhackme.com/room/volatility) - `Memory analysis tool that spans across windows and linux`
5. [Redline](https://tryhackme.com/room/btredlinejoxr3d) - `incident response tool for gathering and analysis of forensic information`
6. [Velociraptor](https://tryhackme.com/room/velociraptorhp) - `endpoint monitoring, forensics, response platform`

In the next article, I will be sharing my learning from [Windows Forensics 1](https://tryhackme.com/room/windowsforensics1)
