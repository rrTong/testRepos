# Project 4 : File system
Melanie Zhang, Ryan Tong <br />
May 30, 2018 <br />
## Introduction
The goal of this project is to create a basic file system. This file system 
takes inspiration from the FAT system architecture. <br />
## Design
Our program utilizes the provided disk.c file. <br />
The main components of the file system is located in `fs.c` <br />
The relevant and unprovided testers that we used are :
* test.txt
* test_fs_stat.sh
* test_fs_read.sh
* test_fs_write.sh
### fs.c
Our file system consists of similar specifications a FAT system architecture 
would have:
* Superblock
* FAT
* Root directory
##### struct Superblock
The Superblock is the first block of the file system, describing what the file 
system is made up of. Our `Superblock` struct consists of 

Data Type | Name | Description
--------- | ---- | -----------
uint8_t[8] | signature | Signature (must be equal to “ECS150FS”)
uint16_t | num_blocks | Total amount of blocks of virtual disk
uint16_t | root_dir_idx | Root directory block index
uint16_t | data_block_idx | Data block start index
uint16_t | num_data_blocks | Amount of data blocks
uint8_t | num_fat_blocks | Number of blocks for FAT
uint8_t[4079] | padding | Unused/Padding
We implement the signature as an array of 8 uint8_t entries, and the padding 
as an array of size 4079 of `uint8_t` entries, in order to perfectly match the 
specs, as well as add the attribute `packed`.
##### FAT
The FAT or File Allocation Table is an array of  entries of type```uint16_t 
*fat;```.
##### Root directory
The root directory is an array of 128 entries, with each entry modeled as a 
struct `Root_dir_entry`, with the members below:

Data Type | Name | Description
--------- | ---- | -----------
uint8_t[FS_FILENAME_LEN] | filename | Filename (including NULL character)
uint32_t | file_size | Size of the file (in bytes)
uint16_t | first_index | Index of the first data block
uint8_t[10] | padding | Unused/Padding
#### fs_mount()
This function initializes and sets up the file system of this virtual disk. 
Information is obtained and extracted according to the diskname that is passed 
in. <br />
We first determine whether there is already an underlying virtual disk that is 
mounted currently (using a global variable that contains the name of the 
virtual disk or NULL if no virtual disk is mounted). We chose to use a global 
variable because almost all of our functions need to check whether there is an 
underlying virtual disk mounted before proceeding. <br />
Using block_read(), we read in the first block of the disk into a char buffer 
of `BLOCK_SIZE`, and then cast it to a pointer to a struct Superblock. We then 
perform some error checking, making sure that the first block's signature is 
indeed `ECS150`, and that the number of blocks and other information match up. 
<br />
We then allocate space for the FAT from the information now stored in the 
superblock, and iterate through each block of the FAT, reading it into a 
buffer and then transferring the buffer's contents into the fat array. Lastly, 
we read the block at the root directory index (as listed in the superblock) 
and transfer the contents to our root directory array. <br />
We chose to use `memcpy` in order to transfer contents from buffers filled by `
block_read` into our data structures because it was the simplest and most 
sensical choice, as it simply copies bytes (with no worry about the type of 
the data, which aligned with the disk functions as they were all data-type 
blind). <br />
#### fs_umount()
This function frees the data structures allocated in `fs_mount` and closes the 
underlying disk file. We first check for open file descriptors (the data 
structures we chose to implement file opening and file descriptors will be 
explained later), making sure there are no currently open file descriptors. <br
 />
We then need to write the changed metadata back to the disk, using calls to `
block_write` in order to store the superblock, FAT, and root directory back to 
the virtual disk. We free the FAT, close the virtual disk name, and set our 
global diskname to NULL, signifying that there is no longer an underlying 
virtual disk mounted. <br />
#### fs_info()
This function, as per the reference program, prints information about the 
filesystem. In order to calculate `fat_free_ratio` and `rdir_free_ratio`, we 
defined two helper functions `get_fat_free` and `get_rdir_free` which iterate 
through our FAT and root directory arrays, counting the number of free entries.
 We chose to implement these as iterative functions instead of storing global 
variables of the counts to avoid cluttering the global space. <br />
#### fs_create()
This function creates a new empty file and adds it to the root directory.We 
iterate through the root directory, looping until we find an index 
corresponding to an empty entry, as per the first-fit approach. We check that 
there is no file with the passed in file name already existing, as well as 
that the root directory is not full. <br />
We then create a new entry for the file (as a `struct Root_dir_entry`), use `
memcpy` to set the `filename` member (again for simplicity and to emphasize 
that the filename is not inherently typed). We also initialize the entry's 
file size as 0, and set the first data block index as `FAT_EOC`. Lastly we set 
the root directory's entry at the free index to be this newly created entry.
#### fs_delete()
This function deletes a specified file from the file system. The root 
directory is scanned until the filename with the matching name is found. To 
indicate that the entry at the current index is empty, we set the first byte 
of the file name to be 0. We then follow the file's chain in the FAT, setting 
each entry to 0 to indicate that it is no longer being allocated by any file. 
The actual data blocks of the disk are not changed, since the FAT already 
reflects that the data blocks are now free to be used by another file. <br />
#### fs_ls()
This function lists information about the files in our root directory. Using a 
for loop, the function goes through the entire root directory, printing the 
file's name `filename_temp`, size `file_size`, and corresponding data block `
first_index`. If the entry is empty, it will not print. <br />
#### fs_open()
This function opens a file. We implemented open files by defining a `struct Fil
e` with members `fd`, `offset`, `root_dir_index`, and `filename`. We needed to 
track the `fd` and `offset` for almost all operations involving open files, 
and we decided to add the root directory index as well as the filename to aid 
in finding this file and locating other pertinent information in the root 
directory in other functions, such as read and write. <br />
In order to track open file descriptors, we used a global array of size 32 of `
struct File` pointers. Since there was a maximum of 32 open files at a time, 
this implementation seemed the most straightforward and natural to us; we 
simply found the file corresponding to a certain file descriptor by indexing 
into the array with the file descriptor, and any non-used file descriptors 
would be `NULL` at their index in the array. We made this array global because 
multiple functions needed access to it. <br />
After error checking (validity of file name, max number of files open, etc), 
we allocate a new `File` object and initialize its member variables. The `fd` 
is  determined using a helper function `find_free_fd()` that simply loops 
through the global array of open files and returns the first index 
corresponding to a null entry. We then update the global array at the index `fd
` to store this `File` pointer. <br />
#### fs_close()
This function closes a file. We locate the `File` object of this file by 
indexing into our global array, set the entry to `NULL`, and free the `File` 
pointer. <br />
#### fs_stat()
This function returns the size of a file. It first locates the file by 
indexing into the global array of currently open files. It then uses the root 
directory index stored in this struct to index into the root directory and 
return the size of the file as listed in the directory. <br />
#### fs_lseek()
This function moves the offset of a file, given a value. With `fd` and `offset`
 passed in, this function locates the file at index `fd` in our global array 
and sets its `offset` member to the provided `offset`, provided the offset is 
not negative or greater than the current size of the file. <br />
#### fs_read()
This function reads a from a file, given a file descriptor id `fd`, the buffer 
to receive the read data `buf`, and the number of bytes to be read `count`. We 
first perform some error checking, and define a variable `actual_count` to 
be the maximum of either the passed in `count` or the difference between the 
file's size and the file's offset (since if `count` is too large, we simply 
read as much as we can). Another important variable we use is `buf_offset` 
which is simply how many bytes we've read into the buffer already, and we 
increase this variable's value each time we read into the buffer to know which 
position to read to next. <br />
We split the reading process into three main parts:
* first block
* full blocks
* last block
##### first block
We first find the block number corresponding to the file's current offset by 
simply dividing the file's offset by `BLOCK_SIZE` (to see how far down the FAT 
chain to go to find the first block to read from corresponding to the offset). 
Then we find the offset within this block to start reading from, which is the 
remainder of the first operation. In order to find the actual index of this 
block within our file system, we use a helper function `get_ith_data_block_inde
x` that simply follows the FAT chain `i` times and returns the correct index (
or -1 if out of bounds). <br />
We then allocate a bounce buffer to read the entire first block into, and 
calculate how much of this block we're actually going to be reading: either 
all of the first block after the offset, or just part of it (depending on the 
value of `actual_count`). We then use `memcpy` to transfer the calculated 
amount of bytes to `buf`, because `memcpy` provides us much finer control than 
`block_read` in order to read in part of a block. We chose to use a bounce 
buffer in order to keep this function as general as possible; whether or not 
we need to read the entire first block or just a part of it, this method will 
still work for either case. <br />
##### full blocks
We then calculate the number of full blocks to be read (dividing the 
difference between `actual_count` and the amount already read by BLOCK_SIZE), 
as well as the number of bytes to read from the last block, if any (which is 
the remainder of the previous operation). Then using a for loop, we simply use 
calls to `block_read` in order to transfer over the contents to our buffer, 
and update `buf_offset` as we go. <br />
##### last block
We then calculate the index of the last block that we need to read from (which 
should be the block `num_whole_blocks` + 1 down the FAT chain) using our 
helper function. If this returns -1, we are done: the file does not contain 
any more blocks. If the index returned is not -1, we again use a bounce 
buffer, read the entire last block into it, and use `memcpy` to copy the 
needed content into the actual buffer. Again, we use a bounce buffer to keep 
this function as general as possible; whether we are to read just a few bytes 
or even no bytes at all from this block, the right amount of bytes will always 
be read into the buffer. <br />
In the above three sections, at times we decided to sacrifice efficiency for 
generality. We could have used many if clauses to address each case and see if 
bounce buffers were needed, but the way we implemented the block reading and 
memory copying operations allowed us to address every single case in a much 
more general way. At the very end we update the file's offset. <br />
#### fs_write()
This function writes to a file, given a file descriptor id `fd`, the contents 
to write into the file `buf`, and the number of bytes to be written `count`. 
This function uses a very similar procedure to `fs_read`, with the main 
differences being that we are using `block_write` for all of our writing 
operations, and that we need to check if the file needs to be extended before 
each write operation. After reading entire blocks into buffers, we simply 
update the parts that need to be written to with `memcpy` (this function best 
matches what we are aiming to do, simply overwriting bytes in memory and 
providing granularity) and write the dirty block back to disk. The index 
calculating and arithmetic is nearly identical to what we used in `fs_read`, 
so it will not be explained here again. <br />
To keep track of how many bytes we've written, we keep a variable `bytes_writte
n` that is incremented after each write operation, which allows us to update 
the buffer position to write the next needed contents. <br />
The process can be separated as 3 parts again:
* first block
* full blocks
* last block
##### first block
After calculating the index of the first block to write to, we extend the file 
if needed using the helper function `allocate_new_data_block` (explained later)
. We first read the entire first block into a temporary buffer, and calculate 
the amount to write to the first block (either the entire block minus the 
offset or just a "slice", depending on the value of `count` passed in). We 
then write content to the temporary buffer using `memcpy` and use `block_write`
 to write the dirty block back to disk, and update `bytes_written`. <br />
##### full blocks
We then calculate the number of full blocks following this first block that 
are to be completely overwritten, the same way that we calculated for `fs_read`
, as well as the number of bytes to write to the last block. Again using a for 
loop, we follow the FAT chain (using our helper function `
get_ith_data_block_index`). If the file needs to be extended, we use `
allocate_new_data_block`. We then write `BLOCK_SIZE` bytes (from the `
bytes_written` position of the buffer), and update `bytes_written`. <br />
##### last block
For the last block, we first check if there is any more content to write (as 
calculated previously). If there is, we check if the file needs to be extended 
and extend if necessary, and then read the entire last block into a temporary 
buffer. We then write the rest of the content into the temporary buffer (from 
the start of the block read) using `memcpy` and write the dirty block back to 
the disk just as we did for the first block. Lastly we update the file offset. 
<br />
##### allocate_new_data_block()
This function extends the FAT chainmap of the passed in file descriptor, 
allocating a new data block in the FAT to the file. After error checking, we 
follow the FAT chain with a for loop to get the index of the last data block 
allocated to this file (or if the file is empty, we never enter the for loop 
and will simply get an index of -1). We then call our helper function `
find_available_fat_block` which simply iterates through the FAT to look for an 
empty index, and check if there is any more space. Then, if the file was an 
empty file, we set its first index in its entry in the root directory to be 
the empty index we found. Otherwise, we simply update the FAT's entry at the 
index of the last data block to hold the empty index, and update the entry at 
the empty index to signify the new end of chain. <br />
### test.txt
This is a text file that we used for testing reading and writing. Its contents 
are "test", as an elementary test of our read and write functions. We changed 
the contents of test.txt to test different byte sizes for different cases 
between different file systems and different testers. <br />
### test_fs_stat.sh
This is a simple shell tester for Phase 3 which runs ```fs_ref.x stat``` and 
```test_fs.x stat```. We set up disk.fs with 8192 bytes and add the test.txt 
file into disk.fs using fs_make.x and fs_ref.x. We then run each test case, 
showing the expected output along with the output using our fs.c. <br />
### test_fs_read.sh
This is a simple shell tester for Phase 4 fs_read() which runs ```fs_ref.x cat
``` and ```test_fs.x cat```. We set up disk.fs with 8192 bytes and add the test
.txt file into disk.fs using fs_make.x and fs_ref.x. Using fs_ref.x and test_fs
.x, we run their cat commands to test our fs_read() with the test program's 
fs_read(). <br />
### test_fs_write.sh
This is a simple shell tester for Phase 4 fs_write() which runs ```fs_ref.x add
``` and ```test_fs.x add```. We set up disk.fs with 8192 bytes. Then we add 
test.txt to disk.fs, using both fs_ref.x add and test_fs.x add to test our fs_
write(). We then use fs_ref.x cat for both, to check if the content of the 
files match. <br />
## Difficulties
Running our fs_read(), initially we started reading from elsewhere instead of 
the data block. We thought block_read wasn't returning the right information.
We used the first data block index of a file as read from the root directory, 
and called block_read with this index, which returned with `????`. This was 
misleading because we didn't understand what was wrong with block_read(), 
since the index that we were using was correct as per our `ls` implementation 
as well as the reference program. <br />
Although we knew it wasn't because of block_read(), we eventually figured out 
through observation and a little Piazza that the block_read() was not reading 
at the start of the data block index. In our helper function get_ith_data_block
_index(), we made sure that the returned value included the additional offset 
that wasn't provided before `data_index + superblock->data_block_index`. <br />
Writing our fs_read() and fs_write() functions was also complicated in some 
cases because it hard to make each function fit an overarching general case. 
Combined with additional helper functions, it was difficult to decide how 
sections of the functions should be organized at first. It took lots of 
thinking and a good pen on paper session to figure out the logic we wanted to 
use. <br />
## References
* project4.html
* [The GNU C Library](https://www.gnu.org/software/libc/manual/html_mono/libc.html)
* Canvas and Piazza
