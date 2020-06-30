# espsyncer

Developer tool for ESP/MicroPython

This is a very simple tool that can synchronize data between your 
local folder and any device running MicroPython (`MP`),
connected with a serial port. It was tested and used with ESP32 and
ESP8266 boards (but it should work with other boards too).

It uses the REPL prompt for communication - it only works if the REPL
prompt is available on your device.

## Installation

This program requires Python 3 and pyserial on your computer, and MicroPython on 
your device, connected with a serial port. This program is a single python file,
that should be put onto your PATH and used as a tool. For this reason, usually
you don't want this to be run in a virtual environment.

To install pyserial globally:

	python -m pip install pyserial
	
In this document, `MP` represents the MicroPython device, "remote file" represents a file stored on 
the connected MicroPython device, and "local file" represents a file stored on the computer that is 
running `espsyncer`.
	
## Wiring

ESP8266 and ESP32 specific USB serial wiring instructions:

* GPIO0 to DTR
* RST to RTS
* RX to TX
* TX to RX

This program also works with NodeMCU boards and alike. (ESP32 and ESP8266 does not require 
the "cross bjt" circuit found on those boards, but it also works with that.)

## Command line parameters

	espsyncer.py [-h] [-v] [-o] [-c] [-q] [-b BAUDRATE] [-t TIMEOUT]
				 [-p PORT] [--output OUTPUT]
				 command [params [params ...]]

		
The following commands are supported:

* `reset` - reset your device (DTR/RTS lines)
* `ls` - list files in a directory on `MP`
* `lsl` - list files and their sizes in a directory on `MP`
* `rm` - remove a file from `MP`
* `mkdir` - create a directory on `MP`
* `makedirs` - create a directory path on `MP` recursively, if not exists
* `rmtree` - remove a directory on `MP`˛recursively
* `upload` - upload a file (from local computer to `MP`)
* `download` - download a file (from `MP` to local computer)
* `execute_file` - execute local file contents on `MP`
* `execute` - execute command on `MP`
* `hot_execute` - execute file with hot reload (see details below)


## Options


	  -v, --verbose         Be verbose
	  -o, --overwrite       Overwrite (for upload/download)
	  -c, --contents        Copy contents of the source directory, instead of the
							source directory itself.
	  -q, --quick           Copy only if file size is different.
	  -b BAUDRATE, --baudrate BAUDRATE
							Baud rate, default is 115200
	  -t TIMEOUT, --timeout TIMEOUT
							Timeout, default is 5. Any non-positive value means
							infinite.
	  -p PORT, --port PORT  Port to be used. Defaults to ESP_PORT environment
							variable.
	  --output OUTPUT       Output file. Messages received from MP will be
							written here. For stdout, use '-'.

## Commands

Warning! All commands will reset your device after connecting.

### Reset

Usage:

	espsyncer.py reset
	
What it does:

	Resets the device by setting DTR and RTS, wait 500msec and then clearing RTS.
	Finally it waits for the MicroPython prompt ">>>" to appear on the serial line.
	
Caveats:

	This command will fail if ">>>" does not appear on the serial line after the reset.
	Some reasons why it can happen:
	
	* Wrong port number or baudrate is selected
	* MicroPython is not installed on the device
	* MicroPython is installed on the device, but serial communication is disabled / does not work
	* MicroPython is installed on the device, but it has a boot.py or main.py file that
	  takes control of the MCU and preventing it from displaying the REPL prompt.

### ls

List files and directories in the given directory and writes that list to stdout. The remote directory must exist.
This will print each name on a separate line. Directory names will end with '/'.

Usage:

	espsyncer.py ls <MP directory>
	
Example:

	espsyncer.py ls /
	
Example output:
	
	boot.py
	webrepl_cfg.py

	
### lsl

List files and directories in the given directory and writes that list to stdout. The remote directory must exist.
This will print each name and its size on a separate line, separated by tab (\t). Directory names will end with '/'.

Usage:

	espsyncer.py lsl <MP directory>
	
Example:

	espsyncer.py lsl /
	
Example output:

	some_directory/	0
	boot.py	228
	connect.py	1153
	run.py	805


### rm

Remove a file. The remote file must exist. You cannot remove directories with this command, only files.

Usage:

	espsyncer.py rm <MP file>
	
Example:

	espsyncer.py rm /boot.py
	
Work in progres...