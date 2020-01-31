# Kaitai Struct: runtime library for C++/STL

This library implements Kaitai Struct API for C++ using STL.

Kaitai Struct is a declarative language used for describe various binary
data structures, laid out in files or in memory: i.e. binary file
formats, network stream packet formats, etc.

# Example Usage

The following is a simplified example of how to use the C++ runtime in your project.  
There is nothing fancy here.  It is kept simple to show the basic steps required.

The data described below is similar to that in a real project.  Data was read
over a socket and put into a buffer of 32-bit values and the buffer passed to one of a set of
functions for interpretation.  The function selected was determined by a value in the
header of the data structure.

In the example below we will not be reading from a socket.  Instead we will write the
data to be parsed to a file and then read that data from the file and parse it. 
This is similar to the documentation found elsewhere and keeps the code simple.

## Problem Statement

We are asked to parse data stored in a binary file.  One of the kinds of data we need 
to reading is structured as a sequence of 32-bit values interpreted as follows:

| 31 | 30 | 29 | 28 | 27 | 26 | 25 | 24 | 23 | 22 | 21 | 20 | 19 | 18 | 17 | 16 | 15 | 14 | 13 | 12 | 11 | 10 | 09 | 08 | 07 | 06 | 05 | 04 | 03 | 02 | 01 | 00 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| <td colspan=8>CmdId=0x01              |   <td colspan=8>RSVD(0)               |  <td colspan=16>EntryNum                                                      |
| <td colspan=16>RSVD(0)                                                        |  <td colspan=16>NumWords |
+
| <td colspan=32> NumEntries |
| <td colspan=32> Entry      |
| <td colspan=32> Entry      |
+

The first two rows are a common header.  The `CmdId` field identifies a specific subtype of 
data element and, therefore, how it is to be parsed.  The value 0x01 specifies the 
elements below the header portion are to be interpreted as shown above.  The 
`NumWords` element specifies the remaining number of 32-bit values to read.  This ends the 
header.

The data for the 0x01 subtype consists of a 32-bit integer giving the number of entries in the
buffer.  This is followed by the values themselves, as sequence of `NumEntries` 32-bit values.

All very simple.  No bit-shifting.  Just parsing binary data.  Let's create a 
kaitai parser for this.  Start by creating a file `msg.ksy` and populate it with the
following:

```
meta:
  id: msg
  endian: le
  imports:
    - header
seq:
  - id: hdr
    type: header
  - id: msg
    type: body
types:
  header:
    seq:
      - id: seq_num
        type: u2
      - id: res0
        type: u1
      - id: cmd
        type: u1
      - id: num_words
        type: u2  
      - id: res1
        type: u2
  body:
    seq:
      - id: num_samples
        type: u4
      - id: data
        type: u4
        repeat: expr
        repeat-expr: num_samples
```

Run `ksc msg.ksy` and you should see two files have been created:  `msg.cpp` and `msg.h`.

Now, create a test application to make use of this parser.  First create the source file
for the application.  Call it `tst.cpp` and populate it with the following:

```c++
#include <iostream>
#include <fstream>
#include <kaitai/kaitaistream.h>
#include "msg.h"
#include "header.h"

using namespace std;

int main(int argc, char**argv)
{
	uint32_t data[] = {
		0x0d000102,  // Header 1
		0x2,         // Header 2
		0x1,         // num_samples
		0x01020304,  // sample_1
	};
	
	ofstream wf("data.dat", ios::out | ios::binary);
	if (!wf) {
		cout << "Cannot open data file for writing" << endl;
		return 1;
	}
	
	for (int ii=0; ii<5; ii++) {
		wf.write((char*)&(data[ii]), sizeof(uint32_t));
	}
	wf.close();
	if (!wf.good()) {
		cout << "Error occurred writing data" << endl;
		return 1;
	}
	
	ifstream is("data.dat", ios::in | ios::binary);
	if (!is) {
		cout << "Cannot open data file for reading" << endl;
		return 1;
	}
	
	kaitai::kstream ks(&is);
	msg_t msg(&ks);
	
	std::cout << "Message Header:" << std::endl;
	std::cout << "  Seq Num:   0x" << std::hex << (int)msg.hdr()->seq_num() << std::endl;
	std::cout << "  Cmd:       0x" << std::hex << (int)msg.hdr()->cmd() << std::endl;
	std::cout << "  Num Words: 0x" << std::hex << (int)msg.hdr()->num_words() << std::endl;
	std::cout << "Message Data:" << std::endl;
	std::cout << "  Num Samples: 0x" << std::hex << (int)msg.msg()->num_samples() << std::endl;
	std::cout << "  Sample[0]:   0x" << std::hex << msg.msg()->data()->front() << std::endl;
}
```

Notice the line `#include <kaitai/kaitaistream.h>` at the top.  This comes from the documentation
on [the C++/STL Notes page](https://doc.kaitai.io/lang_cpp_stl.html).  Try compiling this with:

```
g++ tst.cpp msg.cpp -o tst
```

There are a lot of errors, starting with the compiler being unable to find `kaitai/kaitaistream.h`.
This is because the initial installation of the Kaitai Struct package doesn't include any of the
runtime libraries/include files necessary for building.

To fix this you need to get and build the `kaitai_struct_cpp_stl_runtime` github project and add
it to your application.  The simplest way to do this is to first clone the project (outside of your
test application directory):

```
git clone https://github.com/kaitai-io/kaitai_struct_cpp_stl_runtime kaitai
```

This creates a directory, `kaitai` with a subdirectory also named `kaitai`.  This subdirectory
contains all the code and source files required by the test application.  There is only one 
code file, `kaitaistream.cpp`, and a number of header files, including `kaitaistream.h`.

The easiest way to include this code in your project is to copy that directory into your
project directory so now you have a `kaitai` subdirectory in your project.  Change your
compile command to:

```
g++ tst.cpp -I . msg.cpp kaitai/kaitaistream.cpp -o tst
```

This almost works.  There is a requirement to have defined a preprocessor macro to 
support different string encoders.  For our purposes we can use `KS_STR_ENCODING_NONE`.
So, use the command:

```
g++ -DKS_STR_ENCODING_NONE tst.cpp -I . msg.cpp kaitai/kaitaistream.cpp -o tst
```

You should now have generated the `tst` executable.

Further reading:

* [About Kaitai Struct](https://github.com/kaitai-io/kaitai_struct/)
* [About API implemented in this library](https://github.com/kaitai-io/kaitai_struct/wiki/Kaitai-Struct-stream-API)
