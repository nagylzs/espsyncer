# espsyncer

Developer tool for ESP/MicroPython

This is a very simple tool that can synchronize data between your local folder and any device running MicroPython (`MP`), connected with a serial port. It was tested and used with ESP32 and ESP8266 boards (but it should work with other boards too).

It uses the REPL prompt for communication - it only works if the REPL prompt is available on your device.

It has one unique feature that may not be found in other tools: it can do "live testing" or "hot reload":

* start "hot reload" in the background for a file, that will monitor the file for changes
* open that file in your favourite editor, and make changes
* whenever you save that file, espsyncer will reset the device, upload the changed version and start it

This allows you to write your program and test "as you go".

## Installation

This program requires Python 3 and pyserial ( https://pypi.org/project/pyserial/ ) on your computer, and MicroPython on your device, connected with a serial port. This program is a single python file, that should be put onto your PATH and used as a command line tool. For this reason, usually you will want to install pyserial globally:

	python -m pip install pyserial

It is also possible to run it in a virtual env, but in most cases there is no reason to do it.
	
## Referencing files on your MicroPython device	
	
In this document, `MP` represents the MicroPython device, "remote file" represents a file stored on the connected MicroPython device, and "local file" represents a file stored on the computer that is running `espsyncer`.

Path separator on remote device is slash (`/`). All remote paths must be given with absolute paths 
(on the device), e.g. they must start with a slash `/`.
	
## Wiring

ESP8266 and ESP32 specific USB serial wiring instructions:

* GPIO0 to DTR
* RST to RTS
* RX to TX
* TX to RX

This program also works with NodeMCU boards and alike. (ESP32 and ESP8266 does not require 
the "cross bjt" circuit found on those boards, but it also works with that.)

## Command line parameters

	espsyncer.py [-h] [-v] [-o] [-c] [-q] [-s] [-b BAUDRATE] [-t TIMEOUT]
				 [-p PORT] [--output OUTPUT]
				 command [params [params ...]]

		
The following commands are supported:

* `reset` - reset your device (DTR/RTS lines)
* `ls` - list files in a directory on `MP`
* `lsl` - list files and their sizes in a directory on `MP`
* `mkdir` - create a directory on `MP`
* `makedirs` - create a directory path on `MP` recursively, if not exists
* `upload` - upload a file (from local computer to `MP`)
* `download` - download a file (from `MP` to local computer)
* `rm` - remove a file from `MP`
* `rmtree` - remove a directory on `MP`˛recursively
* `execute_file` - execute local file contents on `MP`
* `execute` - execute command on `MP`
* `hot_reload` - execute file with hot reload (see details below)


## Options

	  -h, --help            show this help message and exit
	  -v, --verbose         Be verbose
	  -o, --overwrite       Overwrite (for upload/download)
	  -c, --contents        Copy contents of the source directory, instead of the
							source directory itself.
	  -q, --quick           Copy only if file size is different.
	  -s, --stop-on-terminator
							Stop on terminator (b'\r\n>>> ')
	  -b BAUDRATE, --baudrate BAUDRATE
							Baud rate, default is 115200
	  -t TIMEOUT, --timeout TIMEOUT
							Timeout, default is 5. Any non-positive value means
							infinite.
	  -p PORT, --port PORT  Port to be used. Defaults to ESP_PORT environment
							variable.
	  --output OUTPUT       Output file. Messages received from MCU will be
							written here. For stdout, use '-'.

## Commands

Warning! All commands will reset your device after connecting. This is because DTR and RTS line are
initially in HIGH state when pyserial opens a port. More information about this can be found here:

	https://pyserial.readthedocs.io/en/latest/pyserial_api.html#serial.Serial.open
	
Basically, this is hardware/OS dependent, and cannot be changed from espsyncer (or any other software).
Espsyncer will reset your device after opening the serial port, in all cases. This ensures a known
device state.

### Reset

Usage:

	espsyncer.py reset
	
What it does: Resets the device by setting DTR and RTS, wait 500msec and then clearing RTS. Finally it waits for the MicroPython prompt ">>>" to appear on the serial line.
	
Caveats: This command will fail if ">>>" does not appear on the serial line after the reset.

Some reasons why it can happen:
	
* Faulty serial connection
* Wrong port number or baudrate is selected	
* MicroPython is not installed on the device
* MicroPython is installed on the device, but serial communication is disabled or does not work
* MicroPython is installed on the device, but it has a boot.py or main.py file that
  takes control of the MCU and preventing it from displaying the REPL prompt.

When there is no response from the device, then you should check your port number, baud rate and MicroPython formware with a terminal program such as RealTerm or putty.

Please note that the device will always be reset when the port is opened (e.g. all other commands
start with an implicit device reset.)

### ls

List files and directories in the given directory and writes that list to stdout. The remote directory must exist. This will print each name on a separate line. Directory names end with '/'.

Usage:

	espsyncer.py ls <MP directory>
	
Example:

	espsyncer.py ls /
	
Example output:
	
	boot.py
	webrepl_cfg.py


### lsl

List files and directories in the given directory and writes that list to stdout. The remote directory must exist. This will print each name and its size on a separate line, separated by tab (\t). Directory names will end with '/'.

Usage:

	espsyncer.py lsl <MP directory>
	
Example:

	espsyncer.py lsl /
	
Example output:

	some_directory/	0
	boot.py	228
	connect.py	1153
	run.py	805

### mkdir

Create a remote directory. The remote directory must not exist.

Usage:

	espsyncer.py [-v] mkdir <MP directory>
	
Example:

	espsyncer.py -v mkdir /some_dir

### makedirs

Create a remote directory structure. The remote directory may already exist. Non-existent directories will be created.

Usage:

	espsyncer.py [-v] makedirs <MP directory>
	
Example:

	espsyncer.py -v makedirs /some_dir/a/b/c

### upload

Upload a file or a directory structure.

Usage:

	espsyncer.py [-v] [-o] [-q] [-c] upload <src> <dst>

The src argument should be a local file or directory. The dst argument is the remote destination directory on your device. It is important to note that the destination is always interpreted as a directory.

The following options are supported:

* `-v` or `--verbose` - log verbose messages (also print little dots to show upload progress)
* `-o` or `--overwrite` - overwrite destination if it already exists
* `-c` or `--contents` - Copy the contents of the source directory, instead of the directory itself.
* `-q` or `--quick` - Copy source file only of the destination has a different size (or does not exist).
	Please note that only the file size is compared, not its contents.

Examples below.

#### Upload a single file into a remote directory

	espsyncer.py -v upload test/test.txt /

#### Upload a single file into a remote directory, overwrite if it exists 

	espsyncer.py -v -o upload test/test.txt /

#### Upload a single file into a remote directory, upload only if size if different, overwrite if it exists

	espsyncer.py -v -o -q upload test/test.txt /

#### Upload a directory named "test" into /

	espsyncer.py -v upload test /

#### Upload a directory named "test" into /, overwrite existing files

	espsyncer.py -v -o upload test /

#### Upload a directory named "test" into /, upload only if size is different, overwrite existing files

	espsyncer.py -v -o -q upload test /

#### Upload contents directory named "test" into /, upload only if size is different, overwrite existing files	

	espsyncer.py -v -o -q -c upload test /

### download

Download a file or a directory structure.

Usage:

	espsyncer.py [-v] [-o] [-q] [-c] download <src> <dst>

The src argument should be a remote file or directory. The dst argument is the local destination directory on your computer. It is important to note that the destination is always interpreted as a directory.

The following options are supported:

* `-v` or `--verbose` - log verbose messages (also print little dots to show upload progress)
* `-o` or `--overwrite` - overwrite destination if it already exists
* `-c` or `--contents` - Copy the contents of the source directory, instead of the directory itself.
* `-q` or `--quick` - Copy source file only of the destination has a different size (or does not exist).
	Please note that only the file size is compared, not its contents.

This command is almost identical to `upload`, only it copies files in the opposite direction. For examples, see the `upload` command.

### rm

Remove a file. The remote file must exist. You cannot remove directories with this command, only files.

Usage:

	espsyncer.py [-v] rm <MP file>
	
Example:

	espsyncer.py rm /test/test.txt

### rmtree

Remove a directory structure recursively. The remote directory must exist.

Usage:

	espsyncer.py [-v] rmtree <MP directory>
	
Example 1, delete /test recursively:

	espsyncer.py -v rmtree /test

Caveats: the rmtree command removes the given directory, with the exception ofthe root directory. The root directory cannot be removed, so the command

	espsyncer.py rmtree /

will remove everything under /, but not the root directory itself.

### execute

Execute a command on your `MP` device.

Usage:

	espsyncer.py [--output <output>] [-t <timeout>] [-s] execute <python_command>

Value for `--output` can be a local file, or you can use dash (`-`) to use stdout. When `--stop-on-terminator` or `-s` is given, then the program will exit when it finds the command prompt `>>>` in the output of the device. The `--timeout` specifies the maximum amount of time to wait. You can give negative value to wait forever (or until the terminator is reached).

Example:

	espsyncer.py --output - --stop-on-terminator execute "print(12)"

This will run the command "print(12)", log all serial communication to the standard output, and exit when the `>>>` prompt is read from the device.

Output for this example:

	paste mode; Ctrl-C to cancel, Ctrl-D to finish
	=== print(12)
	12
	>>> 

### execute_file

Load code on your computer, send and execute on `MP`. This command is very simiar to `execute`, but it takes input from a file instead of a command line argument.

Note: this command does not save the file onto the file system of your `MP` device! It simply sends the contents of the given local file to REPL (in paste mode).

Usage:

	espsyncer.py [--output <output>] [-t <timeout>] [-s] execute_file <local_python_file>

The `local_python_file` should point to a local file path. If you want to use stdin instead of a local file, use `-`. 

### hot_reload

This command is very similar to execute_file, with the following differences:

* You must give a real local file (you cannot use `-` for stdin)
* The last modification time of the given local file will be monitored for changes. Whenever the file is changed, `espsyncer` will reset the device and re-executes the file on `MP`.

Usage:

	espsyncer.py [--output <output>] [-t <timeout>] [-s] hot_reload <local_python_file>

Example:

	espsyncer.py --output - -t -1 hot_reload test/test.py

This will:

* reset `MP`
* send test/test.py to `MP`, and then continuously forward `MP` serial output to stdout
* it will also monitor the local test/test.py file for changes. When it changes, it will restart the whole procedure (reset, send, forward)

The `-t -1` option tells espsycner that it should wait idefinitely (infinite timeout). This is useful when your test file contains your main program ("main loop") that runs indefinitely on the device.

## Other planned features

* It would be good to extend the hot_reload command to monitor a directory structure, and upload changes of all files from that directory. (Instead of just a single file.)
* The --stop-on-terminator option could have a variant where the terminator could be specified by hand.

## Use from a program

The `espsyncer.py` can be used as a Python3 module. It provides the following classes:

* `EspSyncer` - used to communicate with `MP`
* `EspException` - ESP/MicroPython specific exception class

Most commands have a corresponding method in the `EspSyncer` class, but they expect and return python objects instead of "command strings". For example, the EspSyncer.ls() method returns a Python list of file names, instead of printing them to stdout.
