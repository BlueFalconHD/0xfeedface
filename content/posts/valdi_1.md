+++
date = '2025-03-16T21:17:00-06:00'
draft = false
title = 'Exploring Snapchat’s Javascript Runtime Part I: Valdi Modules'
+++

Recently, I have been using Snapchat quite a lot, and I thought it might be interesting to take a peek inside and see how it runs. Firstly, to analyze any app (for iOS or Android) the application file is required. The assets I will be exploring are present in both versions, however my analysis of the main application focuses on the iOS version only.

To obtain application files for iOS applications, use a website that provides decrypted app files. The decrypted application bundle can be further inspected. Inside it, there is a `Payload` directory, which houses the main `.app` file. Inside that 

> **Note:** Acquiring a decrypted copy is necessary to analyze iOS applications, as without it everything looks like a bunch of garbage. Apple uses this strategy of encrypting applications to prevent piracy. If you have a jailbroken device, existing solutions and tweaks are out there to obtain the decrypted file. If you don’t, Google something like “decrypted iOS apps,” which should lead you to a website where you can acquire the decrypted copies without jailbreaking ;)

For android, the process is far simpler. Search up “Snapchat apk” and click on whatever site you find suitable and download the latest version of Snapchat. From there, change the file extension on the downloaded APK from `.apk` to `.zip` and extract the zip file.

Once I obtained the decrypted copy of the application, I opened it up in IDA and let the analysis finish. This took *quite* a long time (but this is reasonable for a ~200mb binary), so while I was waiting I started poking around in the application’s directory, looking for interesting resources or files. Along with typical resources you would find for iOS applications, there were an immense amount of files with the extension `.valdimodule` (not a typo). Trying to open one of them in a normal text editor revealed a bunch of gibberish, which is usually a good indication that the file is in a binary format (meaning that information is encoded raw, rather than using readable ascii/unicode characters).

Cracking the same file open in a hex editor gives.. a similar amount of gibberish. I will skip ahead ~4 hours here, as after the long process of searching for usages of strings like `valdimodule` and others, I found the decoding logic. After another frustratingly long period of time staring at the complex and intricate bit shifts and math being done on the data which I could hardly understand, I decided to give ChatGPT the hexdump of one of the files to see if it could identify the format.

Sure enough, quickly and efficiently it identified the files to be Zstandard compressed data. If you are unaware, compression is a technique used to, well, compress data to be smaller. In some cases it is absolutely necessary but in others, like this one, the size of the data isn’t reduced by much and in some cases is larger than the uncompressed form, so I am unsure why the decision was made.

Here is when I wish I did the smart thing. There is this incredibly useful command line tool that ships with macOS and Linux (and possibly windows, however I haven’t checked) that can identify the type of a file. Most files, like the Zstandard compressed data, video or music files, images, and much more, include a so-called “magic number” at the very beginning of the data. Most of the time it is gibberish or downright silly (the title of this site is the magic number for executable files in the XNU kernel, for example), and in the case of Zstandard it was just some random numbers. The `file` CLI checks the magic number against its database and lets you know what kind of data you have. I should have run the file utility against the `.valdimodule` files, which tells me exactly what four hours of work did in a few milliseconds.

After decompressing some of the files, I faced yet another binary file, this time it wasn’t recognized by the `file` utility, though. I did get somewhat lucky, though, since the files contained readable stuff. In fact, they contained some Javascript.

I threw together some Python code that finds a common sequence at the start of all of the binaries, which can help me determine the magic number far easier and find out where the real data starts. After I determined that, it was just a matter of deciphering the format that the data is in, which is a simple entry-based data structure where each entry has a name and some content. This serves as a sort of archive where multiple files can be packed into one large file and unpacked when needed.

> **Note:** I know this post is kind of short and doesn’t include much detail. I am working on improving my technical writing skills, specifically those of describing my process and exploration in much greater detail. However I am not quite there yet. Because of this, I am providing a link to a python script I wrote that extracts the valdi modules, so if you want more details, check my implementation out here: https://gist.github.com/BlueFalconHD/0b6b582595fe04d594a06ac4dc93a930

After extracting all the Valdi modules, there is some very juicy Javascript code that handles the creation of UI and everything, which is very interesting. I probably would get in trouble if I  provided the contents, so instead use the above script to extract them yourself. For the iOS version, create the directory `valdi` and run the command `python extract_valdi_modukes.py Picaboo\ 13.31.1/Payload/Snapchat.app/*.valdimodule -f -o valdi` to extract the modules.
