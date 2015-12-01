# dynStruct
dynStruct is a tool using dynamoRio to monitor memory access of an ELF binary via a data gatherer,
and use this data to recover structures of the original code.

The data gathered can also be used to quickly find where and by which function a menber of a structure is write or read.

Today only the data gatherer and the structure recovering is available, the web_ui will come.

## Requirements
* CMake >= 2.8
* [DynamoRIO](https://github.com/DynamoRIO/dynamorio)

## Setup
Set the environment variable DYNAMORIO_HOME to the absolute path of your
DynamoRIO installation

execute `build.sh`

to compile dynStruct for a 32bits target on a 64bits os execute `build.sh 32`

## Data gatherer

###Usage

```
drrun -c <dynStruct_path> <dynStruct_args> -- <prog_path> <prog_args>

  -h print this help
  -o <file_name>	set output file name for json (default: <prog_name>.ds_out)
   			print output on console
  -w <module_name>	wrap <module_name>
			 dynStruct record memory blocks only
			 if *alloc is called from this module
  -m <module_name>	monitor <module_name>
			 dynStruct record memory access only if
			 they are done by a monitore module
  -a <module_name>	is used to tell dynStruct which module implements
			 allocs functions (malloc, calloc, realloc and free)
			 this has to be used with the -w option (ex : "-a ld -w ld")
			 this option can only be used one time
for -w, -a and -m options modules names are matched like <module_name>*
this allow to don't care about the version of a library
-m libc.so match with all libc verison

The main module is always monitored and wrapped
Tha libc allocs functions are always used (regardless the use of the -a option)

Example : drrun -c dynStruct -m libc.so - -- ls -l

This command run "ls -l" and will only look at block allocated by the program
but will monitor and record memory access from the program and the libc
and print the result on the console
```

### Example

We are going to analyse this little program.

```C
void print(char *str)
{
  puts(str);
}

int main()
{
  char *str;

  str = malloc(5);
  strcpy(str, "test");
  str[4] = 0;
  print(str);

  free(str);
}
```
Which after compilation look like this
![Example disassembly](http://i.imgur.com/L2i4zJS.png)

If we run `drrun -c  dynStruct - -- tests/example` we get
```
test
tast
block : 0x0000000000603010-0x0000000000603015(0x5) was free
alloc by 0x0000000000400617(main : 0x00000000004005f9 in example) and free by 0x000000000040064c(main : 0x00000000004005f9 in example)
	 WRITE :
	 was access at offset 1 (1 times)
	details :
			 1 bytes were accessed by 0x00000000004005e7 (print : 0x00000000004005b6 in example) 1 times
	 was access at offset 4 (2 times)
	details :
			 1 bytes were accessed by 0x000000000040062a (main : 0x00000000004005f9 in example) 1 times
			 1 bytes were accessed by 0x0000000000400636 (main : 0x00000000004005f9 in example) 1 times
	 was access at offset 0 (1 times)
	details :
			 4 bytes were accessed by 0x0000000000400624 (main : 0x00000000004005f9 in example) 1 times
```
We see all the right access on str done by the program himself.
We can notice the 4 bytes access at offset 0 of the block due to gcc optimisation for initializing the string.

Now if we run `drrun -c  dynStruct -m libc - -- tests/example` we are going to monitor all the libc access, and we get
```
test
tast
block : 0x0000000000603010-0x0000000000603015(0x5) was free
alloc by 0x0000000000400617(main : 0x00000000004005f9 in example) and free by 0x000000000040064c(main : 0x00000000004005f9 in example)
	 READ :
	 was access at offset 1 (2 times)
	details :
			 1 bytes were accessed by 0x00007fca292156c2 (_IO_default_xsputn : 0x00007fca29215650 in libc.so.6) 1 times
			 1 bytes were accessed by 0x00007fca29213c9d (_IO_file_xsputn@@GLIBC_2.2.5 : 0x00007fca29213b70 in libc.so.6) 1 times
	 was access at offset 2 (2 times)
	details :
			 1 bytes were accessed by 0x00007fca292156c2 (_IO_default_xsputn : 0x00007fca29215650 in libc.so.6) 1 times
			 1 bytes were accessed by 0x00007fca29213c9d (_IO_file_xsputn@@GLIBC_2.2.5 : 0x00007fca29213b70 in libc.so.6) 1 times
	 was access at offset 3 (2 times)
	details :
			 1 bytes were accessed by 0x00007fca292156c2 (_IO_default_xsputn : 0x00007fca29215650 in libc.so.6) 1 times
			 1 bytes were accessed by 0x00007fca29213c81 (_IO_file_xsputn@@GLIBC_2.2.5 : 0x00007fca29213b70 in libc.so.6) 1 times
	 was access at offset 0 (5 times)
	details :
			 1 bytes were accessed by 0x00007fca292156c2 (_IO_default_xsputn : 0x00007fca29215650 in libc.so.6) 1 times
			 16 bytes were accessed by 0x00007fca292210ca (strlen : 0x00007fca292210a0 in libc.so.6) 2 times
			 4 bytes were accessed by 0x00007fca29224c2e (__mempcpy_sse2 : 0x00007fca29224c00 in libc.so.6) 1 times
			 1 bytes were accessed by 0x00007fca29213c9d (_IO_file_xsputn@@GLIBC_2.2.5 : 0x00007fca29213b70 in libc.so.6) 1 times
	 WRITE :
	 was access at offset 1 (1 times)
	details :
			 1 bytes were accessed by 0x00000000004005e7 (print : 0x00000000004005b6 in example) 1 times
	 was access at offset 4 (2 times)
	details :
			 1 bytes were accessed by 0x000000000040062a (main : 0x00000000004005f9 in example) 1 times
			 1 bytes were accessed by 0x0000000000400636 (main : 0x00000000004005f9 in example) 1 times
	 was access at offset 0 (1 times)
	details :
			 4 bytes were accessed by 0x0000000000400624 (main : 0x00000000004005f9 in example) 1 times

```
Now all the read access done by the libc are listed.

## Structure recovery

The python script dynStruct.py do the structure recovery and will start the web_ui when available.

The idea behind the structure recovery is to have a quick idea of the structures are used by the program.

It's impossible to recover exactly the same structures than it was in the source code, so some choice were made.
To recover the size of members dynStruct.py look at the size of the accesses for a particular offset, it keep the more used
size, if 2 or more size are used the same number of time it keep the bigger size.

All type are ```uint<size>_t```, all the name are ```offset_<offset_int_the_struct>```.
Some offset in blocks have no access in the ouput of the dynStruct dynamoRIO client, so the empty offset are fill
with array called ```pad_offset<offset_in_the_struct>```, all pading are uint8_t.
Array are detected, 5 or more consecutive members of a struct with the same size is considered as an array.
Array are named ```array_<offset_int_the_struct>```.
The last thing who is detected is array of structure, they are named ```struct_array_<offset_in_the_struct>```

The recovery of struct try to be the most compact as possible, the output will look like :
```
struct struct_14{
	uint32_t array_0x0[5];
	struct {
		uint64_t offset_0x0;
		uint8_t offset_0x8;
		uint8_t pad_offset_0x9[7];
	}struct_array_0x14[2];
} 
```

### Usage
```
usage: dynStruct.py [-h] [-d DYNAMO_FILE] [-p PREVIOUS_FILE] [-o OUT_PICKLE]
                    [-e <file_name>] [-c] [-w] [-l BIND_ADDR]

Dynstruct analize tool

optional arguments:
  -h, --help        show this help message and exit
  -d DYNAMO_FILE    output file from dynStruct dynamoRio client
  -p PREVIOUS_FILE  file generated by a previous use of the web ui
  -o OUT_PICKLE     name of the file generate by web ui
  -e <file_name>    export structures in C style on <file_name>
  -c                print structures in C style on console
```
### Example
```
drrun -c dynStruct -o out_test -- tests/test
python3 dynStruct.py -d out_test  -c
```
will display
```
//total size : 0x20
struct struct_2 {
	struct  {
		uint32_t offset_0x0;
		uint32_t offset_0x4;
	}struct_array_0x0[2];

	uint64_t offset_0x10;
	uint8_t offset_0x18;
	uint8_t pad_offset_0x19[7];
};

//total size : 0x18
struct struct_3 {
	uint64_t offset_0x0;
	uint64_t offset_0x8;
	uint64_t offset_0x10;
};
```
The array of structure of 2 uint32_t is because dynStruct find only array with a size of 5 or more, so when arrays of structures or search a pattern is found here and every pattern is consider as a array of structures.

The same output can be obtained with :
```
python3 dynStruct.py -d out_test  -o serialize_test
python3 dynStruct.py -p serialize_test -c
```
