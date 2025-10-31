# flash-at-firmware-to-esp-01
Flash a modern version of AT-firmware to a legacy 1-MB ESP-01 module

![Photo of an ESP-o1 module mounted in a dedicated USB programming adapter](images/ESP-01_Programmer.jpg)

This article rediscovers a method I have used to install the AT-command firmware onto old-model ESP-01 modules having just 1 MB of flash.

An ESP-01 module needs to connect through some sort of go-between device for programming. The photo shows one mounted on a USB adapter for direct connection to a computer. 

A switch on the adapter can place the ESP-01 into "Program mode" or "UART Mode". 

Arduino IDE can upload programs onto an ESP-01 connected by a programmer such as the one pictured above. Its Serial Monitor can communicate with the ESP-01 through the UART (another name for a serial interface).

The AT-commands transform the ESP-01 into an interactive appliance having many uses, without writing a program for its built-in ESP8266 processor. For example, the module's WiFi capability can give internet access to projects running on Arduino Unos and Nanos, and on bare AVR microcontrollers.

It may be that Espressif, the manufacturer, supplies an 'official' firmware-flashing application that runs exclusively on the Windows operating system. Because I live in Linux-land, including Mac OS, I needed another way to do it.

One answer is to use other tools and software that Espressif also provides. They include the *esptool.py* utility program and a reasonably up-to-date set of binary files for the firmware.

Tutorials abound online for doing this. Some are obsolete, some feel incomplete, some want the reader to download files from unoffical sources. Many examples proved difficult for me to follow. 

Slowly I gathered scattered information into a plan I could understand and execute. It uses only official, contemporary resources from Espressif, device manufacturer.

I will explain where I found what I needed first. After that I will show I used them.

## Resource Locations

### *esptool.py*


The easiest way I found to install the *esptool.py* utility was to add the ESP8266 board package to my Arduino IDE.

Follow the instructions for *Installing With Boards Manager*, given on the GitHub page for the 8266 Boards package: [https://github.com/esp8266/Arduino](https://github.com/esp8266/Arduino).

*esptool.py* gets installed as part of the ESP8266 package. Arduino IDE uses it to upload programs onto ESP devices. 

Farther down in this article, I will explain how to locate it.


### AT-command Firmware Files


A reasonably current set of files for the AT-command firmware was available in late October, 2025, in a repository that Espressif maintains on GitHub: [https://github.com/espressif/ESP8266_NONOS_SDK](https://github.com/espressif/ESP8266_NONOS_SDK). 

Please note that Espressif discourages use of this and other, older "NONOS" versions of its firmware. It makes sense, as newer hardware products may benefit from newer versions that cannot run on older hardware. 

Meanwhile, however, the older firmware still works well on older hardware such as my 1MB ESP-01 module. 

It appears that Espressif does maintain its final, NONOS version, by fixing bugs, although without adding new features. Language in the repository explains the company's current intentions regarding it.

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

Make a note of where to find this information. You will be entering some of into the command line for the *esptool.py* utility.

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

This program does nothing except to halt operation of the CPU. It is probably not necessary. Uploading a program from the IDE does at least reassure me that the ESP-01 is properly connected and prepared to accept a firmware upload.

<blockquote>
If Arduino IDE complains that it cannot find &#x0027;python&#x0027;, as it did when I ran the IDE under the Ubuntu 24.04 operating system, then you might need to install a short utility package.
 
Here is a link explaining this utility, which is named <a href="https://ubuntu.pkgs.org/25.04/ubuntu-main-amd64/python-is-python3_3.13.3-1_all.deb.html">python-is-python3</a>.
</blockquote>

<h2 id="path-to-epstool">Obtain the Path to the <em>esptool.py</em> Utility</h2>

The *esptool.py* utility runs on the command line of a terminal. The command begins by stating the path to the command. Now, where is it?


I found mine by searching the usually-hidden file folder named '.arduino15'. Here is what I found (on an Ubuntu 24.04 system): 

~~~
/home/iowadave/.arduino15/packages/esp8266/hardware/esp8266/3.1.2/tools/esptool/esptool.py
~~~

That whole string serves as the name of the program.  Terminal commands always begin with the name of the program to be run. Make a note of it. 

I usually copy it and paste it into a text editor. Then I navigate the terminal into the 'bin' folder containing the files to be uploaded. 

It helps to see the file names as I build the rest of the command in the text editor.

Also, the completed command will be copied from the text editor and pasted into the terminal for execution while the terminal is in this 'bin' folder. 

<h2 id="build-the-esptool-command">Build the <em>esptool.py</em> Command</h2>

The command path, obtained in a previous step, needs to be followed by a number of options. Here is the completed command that has given me good results with the 1MB ESP-01 module.

~~~ bash
~/.arduino15/packages/esp8266/hardware/esp8266/3.1.2/tools/esptool/esptool.py \
--chip auto \
--port /dev/ttyUSB0 \
--baud 115200 \
--before default_reset \
--after hard_reset \
write_flash -z --flash_mode keep \
--flash_freq keep --flash_size detect \
0x00000 boot_v1.7.bin \
0x01000 at/512+512/user1.1024.new.2.bin \
0xfc000 esp_init_data_default_v08.bin \
0x7e000 blank.bin \
0xfe000 blank.bin
~~~

Each of the option names begins with two dash characters. The name is followed by a space character then by the value being given to that option. The options instruct *esptool.py* as follows:

* --chip auto | let *esptool.py* determine the type of chip it is flashing
* --port /dev/ttyUSB0 | the hardware address, on my system, of the USB adapter for the ESP-01 module. Readers will need to find this out for themselves on their own systems.
* --baud 115200 | the default baud rate for ESP devices, but stated here anyway for the sake of completeness
* --before default_reset | and the next one...
* --after hard_reset     | are steps the tool should perform before and after flashing. See the online documentation for details. A link to the documentation is given below.
* write_flash | commands the esptool to upload files into the chip's flash memory
--flash_mode keep | the default setting, uses a value given in the firmware
--flash_freq keep | default, uses a value given in the firmware, most likely 40m, indicating a flash memory clock frequence of 40 MHz.
--flash_size detect | try to determine flash size automatically
* 0x00000 boot_v1.7...bin | upload this file to memory location 0x00000
* 0x01000 at/512...bin    | upload this file to memory location 0x01000
* ... and three more file uploads, to their respective memory locations 

Review the listing of file names and addresses copied from the *README.md* file, shown above. Study how the list of options to the utility makes use of the information Espressif provides there.

The README list is specified for a flash size of 8 Mb, meaning 8 mega*bits*. There being 8 bits in a byte, that size equates to 1 MB, mega*byte*.

The size specification is the reason I selected this particular list of file names and addresses to fill out the options list for my *esptool.py* command to flash a 1 MB device.

## Run the Command
Move the switch on the USB adapter to the "Prog" position, putting the ESP-01 into "program mode".

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

If so, you are in business. Notice that the firmware is Version 1.7.6.0 dated January 2022 and the binary files uploaded to the ESP-01 were compiled in June of 2024.

## Helpful Links
I keep the following two links bookmarked on my system for ready reference.

* [*esptool* Reference by Espressif](https://docs.espressif.com/projects/esptool/en/latest/esp32/esptool/index.html)
* [AT-Command Reference](https://docs.espressif.com/projects/esp-at/en/latest/esp32/AT_Command_Set/Basic_AT_Commands.html)

Exercise care with these references: almost certainly they best describe different, more modern hardware compared to my old, ESP-01 modules. Even so, I find them useful.

The esptool reference explains other values that may be tried for flash mode, speed and size in the event that default values do not work as expected. I have not encountered any problem like this with my old ESP-01 modules.

I elected to retain a local copy of that NONOS firmware repository as a safeguard against Espressif deciding to withdraw public access in the future. It is published under a form of open-source license, which I believe authorizes me to copy it. Readers will have to consult their own counsel regarding how the license applies to their individual situations. 
