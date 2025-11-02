# Flash AT-command Firmware To ESP-01
Install an updated, compatible version of AT-firmware to a legacy 1-MB ESP-01 module

![Photo of an ESP-o1 module mounted in a dedicated USB programming adapter](images/ESP-01_Programmer.jpg)

This article rediscovers a method I have used to install the AT-command firmware onto old-model ESP-01 modules having just 1 MB of flash.

It uses recent versions of software and firmware available from &ldquo;official&rdquo; sources, identified below.

## Connecting the ESP-01

An ESP-01 module needs to connect through some sort of go-between device for programming. The photo shows one mounted on a USB adapter for direct connection to a computer. 

A switch on the adapter can place the ESP-01 into "Program mode" or "UART Mode". Both modes are demonstrated in this article. 

When ESP-01 is in program mode, Arduino IDE can upload programs onto it. The other, UART mode enables the Serial Monitor of the IDE to communicate with a program running on the ESP-01 through its serial interface.

The AT-commands transform the ESP-01 into an interactive appliance having many uses, without writing a program for its built-in ESP8266 processor. 

For example, projects running on Arduino Unos and Nanos, and even on bare AVR microcontrollers, can communicate across a WiFi network by sending AT-commands to the ESP-01 and receiving data in return, through its UART serial interface.

It may be that Espressif, the manufacturer, supplies an 'official' firmware-flashing application that runs exclusively on the Windows operating system. Because I live in Linux-land, including Mac OS, I needed another way to do it.

One answer is to use other tools and software that Espressif also provides. They include the *esptool.py* utility program and a reasonably up-to-date set of binary files for the firmware.

Tutorials abound online for doing this. Some feel incomplete, as though expecting me to know more than I do. Some guide the reader to download files from unoffical sources. Many examples proved difficult for me to follow. 

Slowly I gathered scattered information into a plan I could understand and execute. It relies upon community-vetted software from within the Arduino IDE and official firmware obtained from Espressif, the device manufacturer.

First, I will explain where I found the resources I needed. Then I will show how I used them.


## Resource Locations


### *esptool.py*


The easiest way I found to install the *esptool.py* utility was to add the ESP8266 board package to my Arduino IDE.

Follow the instructions for *Installing With Boards Manager*, given on the GitHub page for the 8266 Boards package: [https://github.com/esp8266/Arduino](https://github.com/esp8266/Arduino).

*esptool.py* gets installed as part of the ESP8266 package. Arduino IDE uses it to upload programs onto ESP devices. 

The program will be installed deep-down inside the directory structure of the Arduino IDE's ESP8266 board package. Later in this article, I will explain how to locate it, then how to invoke the esptool for flashing AT-firmware onto the ESP-01.


### AT-command Firmware Files


A recent set of files for the AT-command firmware, suitable for the 1-MB-sized ESP-01 module, was available in late October, 2025, in a repository that Espressif maintains on GitHub: [https://github.com/espressif/ESP8266_NONOS_SDK](https://github.com/espressif/ESP8266_NONOS_SDK). 

Please note that Espressif discourages use of this and other, older "NONOS" versions of its firmware. It makes sense from their point of view, as newer hardware products may benefit from new features introduced in more recent, expanded versions of the firmware that do not fit into or run on older hardware. 

Meanwhile, however, AT-commands designed for the 1-MB ESP-01 still serve my purposes well. I appreciate that Espressif has been keeping the firmware available. 

Moreover, it appears that Espressif does maintain its final, NONOS version by fixing bugs, although without adding new features. Language in the repository explains the company's current intentions regarding it.

Navigate into the 'bin' folder of the repository. If all goes well you may see a list of folders and files similar to the following:

<pre>
at
at_sdio
blank.bin
boot_v1.2.bin
boot_v1.6.bin
boot_v1.7.bin
esp_init_data_default_v05.bin
esp_init_data_default_v08.bin
</pre>

Peek inside the 'at' folder. Notice the README.md file along with a sub-folder named '512-512'. Look inside that sub-folder for two files having names that begin with 'user'. 

If you see all of these files, then you want to download this repository.

Navigate back to the top level, *ESP8266_NONOS_SDK*, click the CODE button and select the option to download a .zip file. 

Unzip the file on your computer, then locate the resulting folder. It will have the same name as the .zip file. Navigate into that folder, where you will see the files from the GitHub repository duplicated on your own computer.

The README.md file inside the 'bin/at' folder gives essential information needed for successful flashing of the firmware. It identifies which files to upload, along with the correct addresses for where to store them in the ESP-01 memory.

Make a note of where to find this information. You will be entering some of it into the command line for the *esptool.py* utility.

<pre>
# BOOT MODE
## download

### Flash size 8Mbit: 512KB+512KB
    boot_v1.2+.bin              0x00000
    user1.1024.new.2.bin        0x01000
    esp_init_data_default.bin   0xfc000
    blank.bin                   0x7e000 & 0xfe000
</pre>

## Prepare the ESP-01

I like to begin by quieting the ESP-01. I want to avoid having any program running or using the serial interface when I attach the USB adapter to the computer.

The way I achieve this is to upload a very short program from the Arduino IDE. Here is the listing:

~~~ c
1 void setup () {}
2 void loop () {}
3 int main () { return 0; }
~~~

Move the switch on the USB adapter into the 'Prog" position and insert the adapter into the computer. Enter the short program into the Arduino IDE. Choose "Generic ESP8266 Module" as the boart type and select the Port where the computer mounted the adapter. My Debian-based Linux system mounted it at /dev/ttyUSB0, for example. 

Make a note of this mount-point, as you will need to enter it on the esptool command line when flashing the AT-command firmware.

The program does nothing except to halt operation of the CPU. It is probably not necessary to do this; however, uploading a program from the IDE does bring some benefits. It reassures me that the ESP-01 is properly connected and prepared to accept a firmware upload.

<blockquote>
<p>If Arduino IDE complains that it cannot find &#x0027;python&#x0027;, as it did when I ran the IDE under the Ubuntu 24.04 operating system, then you might need to install a small utility package named <a href="https://ubuntu.pkgs.org/25.04/ubuntu-main-amd64/python-is-python3_3.13.3-1_all.deb.html">python-is-python3</a>. The link goes to an explanation of what the package does and how it solves the problem.</p>
<p>Install the package in the way your computer normally does. For example, using a Debian Linux-based operating system such as Ubuntu or Raspberry Pi OS,</p>
<pre>

sudo apt update
sudo apt install python-is-python3

</pre> 
</blockquote>

After uploading the program onto the ESP-01 module, quit the Arduino IDE. This step ensures that it stops using the port where the ESP is mounted. Otherwise, if you allow the IDE to keep the port in a 'busy' state, it may interfere with flashing the AT-command firmware at the next step.

<h2 id="path-to-epstool">Obtain the Path to the <em>esptool.py</em> Utility</h2>

The *esptool.py* utility runs on the command line of a terminal. The command begins by stating the path to the command. Now, where is it?


I found mine by searching the usually-hidden file folder named '.arduino15'. Here is what I found (on an Ubuntu 24.04 system): 

~~~
/home/iowadave/.arduino15/packages/esp8266/hardware/esp8266/3.1.2/tools/esptool/esptool.py
~~~

This can be made generic for any user on my computer by replacing the */home/iowadave* prefix with the tilde character. It becomes

~~~
~/.arduino15/packages/esp8266/hardware/esp8266/3.1.2/tools/esptool/esptool.py
~~~

That whole string serves as the name of the program. Terminal commands always begin with the name of the program to be run. Make a note of it. 

I usually copy it and paste it into a text editor. Then I navigate the terminal into the 'bin' folder containing the files to be uploaded. 

It helps to see the file names as I build the rest of the command in the text editor.

Also, the completed command will be copied from the text editor and pasted into the terminal for execution while the terminal is in this 'bin' folder. 

<h2 id="build-the-esptool-command">Build the <em>esptool.py</em> Command</h2>

The command path, obtained in a previous step, needs to be followed by a number of options. Here is an example of a completed command that has given me good results with the 1MB ESP-01 module.

~~~ bash
~/.arduino15/packages/esp8266/hardware/esp8266/3.1.2/tools/esptool/esptool.py --chip auto --port /dev/ttyUSB0 --baud 115200 --before default_reset --after hard_reset write_flash -z --flash_mode keep --flash_freq keep --flash_size detect 0x00000 boot_v1.7.bin 0x01000 at/512+512/user1.1024.new.2.bin 0xfc000 esp_init_data_default_v08.bin 0x7e000 blank.bin 0xfe000 blank.bin
~~~

I separate the options and commands below to aid readability. As you may know, the backslant character at the end of a line of text in a command-line terminal indicates that the text continues on the next line. In other words, what follows below is just another way to write the command shown above.
 

~~~ bash
~/.arduino15/packages/esp8266/hardware/esp8266/3.1.2/tools/esptool/esptool.py \
--chip auto \
--port /dev/ttyUSB0 \
--baud 115200 \
--before default_reset \
--after hard_reset \
write_flash -z 
--flash_mode keep \
--flash_freq keep 
--flash_size detect \
0x00000 boot_v1.7.bin \
0x01000 at/512+512/user1.1024.new.2.bin \
0xfc000 esp_init_data_default_v08.bin \
0x7e000 blank.bin \
0xfe000 blank.bin
~~~

### What Each Option Signifies

Each of the option names begins with two dash characters. The name is followed by a space character then by the value being given to that option. The options instruct *esptool.py* as follows:

* --chip auto | let *esptool.py* determine the type of chip it is flashing
* --port /dev/ttyUSB0 | the hardware address, on my system, of the USB adapter for the ESP-01 module. Readers will need to find this out for themselves on their own systems.
* --baud 115200 | the default baud rate for ESP devices, but stated here anyway for the sake of completeness
* --before default_reset | this option and the next one...
* --after hard_reset     | are steps the tool should perform before and after flashing. See the online documentation for details. A link to the documentation is given below.
* write_flash | commands the esptool to upload files into the chip's flash memory. This is a command, not an option, which is why it does not have the two dash characters preceding it. (I do not (yet) know what the -z option means. It appears i many (but not all) examples found online. It seems to do no harm, at least.)
* --flash_mode keep | the default setting, uses a value given in the firmware, most likely &#x0027;dio&#x0027;, a certain way of wiring the device for uploads.
* --flash_freq keep | default, uses a value given in the firmware, most likely 40m, indicating a flash memory clock frequency of 40 MHz.
* --flash_size detect | try to determine flash size automatically
* 0x00000 boot_v1.7...bin | upload this file to memory location 0x00000
* 0x01000 at/512...bin    | upload this file to memory location 0x01000
* ... and three more file uploads, to their respective memory locations. These file specifications form part of the write_flash command, which is why they are not preceded by dash characters. 

As the example demonstrates, multiple files can be uploaded to different areas in the flash memory during a single run of the esptool. 

Review the listing of file names and memory addresses given in the *bin/at/README.md* file of the firmware repository, shown above. Study how the list of options on the esptool command line makes use of the information Espressif provides there.

The README list is specified for a flash size of 8 Mb, meaning 8 mega*bits*. There being 8 bits in a byte, that size equates to 1 MB, mega*byte*.

The fact that the size specification for these firmware files matches the flash-memory size of my ESP-01 module is the reason I selected this particular list to inform my *esptool.py* command.

## Run the Command

*Important! The command as-written for this example is designed to be executed in a command-line terminal that is open to the 'bin' folder of the user's local copy of the firmware repository.*

Double-check that the switch on the USB adapter is in the "Prog" position, putting the ESP-01 into "program mode".

Copy the completed command from the text editor and paste it into the terminal. Press the Enter key. When all goes well, I see the following output.

~~~
esptool.py v3.0
Serial port /dev/ttyUSB0
Connecting....
Detecting chip type... ESP8266
Chip is ESP8266EX
Features: WiFi
Crystal is 26MHz
MAC: a4:cf:12:ef:ce:c4
Uploading stub...
Running stub...
Stub running...
Configuring flash size...
Auto-detected Flash size: 1MB
Flash params set to 0x0020
Compressed 4080 bytes to 2936...
Wrote 4080 bytes (2936 compressed) at 0x00000000 in 0.3 seconds (effective 124.0 kbit/s)...
Hash of data verified.
Compressed 396900 bytes to 276959...
Wrote 396900 bytes (276959 compressed) at 0x00001000 in 24.4 seconds (effective 130.2 kbit/s)...
Hash of data verified.
Compressed 128 bytes to 75...
Wrote 128 bytes (75 compressed) at 0x000fc000 in 0.0 seconds (effective 78.3 kbit/s)...
Hash of data verified.
Compressed 4096 bytes to 26...
Wrote 4096 bytes (26 compressed) at 0x0007e000 in 0.0 seconds (effective 3896.4 kbit/s)...
Hash of data verified.
Compressed 4096 bytes to 26...
Wrote 4096 bytes (26 compressed) at 0x000fe000 in 0.0 seconds (effective 3836.2 kbit/s)...
Hash of data verified.

Leaving...
Hard resetting via RTS pin...
~~~

"Hash of data verified." indicates to me that the upload succeeded. Let us check.

## Test It in the Arduino Serial Monitor

Remove the USB adapter from the computer. Move the switch on the adapter to the "UART" position, enabling serial communications between the ESP-01 and the computer. Insert the adapter back into one of the computer's USB ports.

Start the Arduino IDE. Select a suitable type of 8266 board. I find the "Generic ESP8266 Module" works well for this purpose.

Select the port corresponding to the ESP programmer; for example, /dev/ttyUSB0.

Start the Serial Monitor of the IDE. Configure it to operate at 115200 baud, and to use "both NL & CR" line ending.

Type <code>AT+GMR</code> on the message line of the Serial Monitor and press Enter. This AT-command instructs the device to identify its firmware. If all has gone well, you might receive back from the ESP-01 some lines of text like the following:

<pre>
AT+GMR
AT version:1.7.6.0(Jan 24 2022 08:56:02)
SDK version:3.0.6-dev(072755c)
compile time:Jun 17 2024 07:38:00
Bin version(Wroom 02):1.7.6
OK
</pre>

If so, you are in business. How nice it was, in year 2025, to see that the firmware Version 1.7.6.0 is dated January 2022 and the binary files uploaded to the ESP-01 were compiled in June of 2024.

## Helpful Links
I keep the following two links bookmarked on my system for ready reference.

* [*esptool* Reference by Espressif](https://docs.espressif.com/projects/esptool/en/latest/esp32/esptool/index.html)
* [AT-Command Reference](https://docs.espressif.com/projects/esp-at/en/latest/esp32/AT_Command_Set/Basic_AT_Commands.html)

## Situational Awareness

Exercise care with these references: almost certainly they best describe different, more modern hardware compared to my old, ESP-01 modules. Even so, I find them useful.

The esptool reference explains other values that may be tried for flash mode, speed and size in the event that default values do not work as expected. I have not encountered any problem like this with my old ESP-01 modules.

I elected to retain a local copy of that NONOS firmware repository as a safeguard against Espressif deciding to withdraw public access in the future. It is published under a form of open-source license, which I believe authorizes me to copy it. Readers will have to consult their own counsel regarding how the license applies to their individual situations. 
