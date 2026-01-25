# CS354 Fall 2025 Final Review 
_Note:This is a direct transfer of notes from my notes app to here to save it forever. I will update the markdown soon to make this more accessible to everyone._

## 1. Multicore Systems - 
- Each core has its own L1 cache 
- Core 0 powered on during boot up, others powered on as req 
- Internal bus to coordinate between variables
- Test and Set used for synchronization 
- Affinity scheduling = tying processes and devices (to handle interrupts by only one core) to specific cores for cache efficiency. When core becomes free, choose thread from affinity set 

## 2. Initialization - 
- Firmware - has the code to initialize the hardware CPU RAM etc
- Before boot - just a binary file on my compile machine. At boot, loaded into BBB’s RAM by U-boot and then executes on BBB’s CPU
- The OS is not the first piece of software that runs = bootstrap program 
- Bootstrap location problem - OS must start at physical location 0
    - Self relocating - must change all addresses
    - Bootstrap is compiled and linked to use addresses beyond OS (easier)
- Steps:
    - Initialize Memory Management 
    - Initialize each OS module
    - Load device driver for each device
    - Start each I/O device
    - Sequential -> concurrent 
    - Create null proc
- Start.S -> null user -> sysinit 
- Sysinit - memory mgmnt hardware and freelist, initializes OS modules, transforms seq to concurrent, enables interrupts 
- Create new process to execute main 

## 3. Subsystem Initialization/Memory marking - 
- Module - lwk anything that has init 
- Global var for initialization int32 needinit = 1, set to 0 when initialized
- Problem = multiple processes can call module functions concurrently -> multiple processes can call the initialization function concurrently -> mutual exclusion
- <img width="741" height="253" alt="Pasted Graphic" src="https://github.com/user-attachments/assets/99f7d6f3-c698-4817-9c60-bf4bd2cf4081" />

￼
- REBOOT neeinit will still be 1
- Accession numbers - modinit < boot -> initialize
- Memmark L - require each module to declare a location to be used 
- Mark(L) notmarked(L)
- The memory marking system guarantees that notmarked(L) will return 1 after the operating system is restarted until mark(L) is called
- Memmark is an array of a single integer 
    * extern int32 *(marks[]);  
    * extern int32 nmarks; 
    * typedef int32 memmark[1]
- Markinit called each time the system reboots 

## 4. Exception Handling 
- Panic - disable interrupts while true busy loop forever 
- Process P is running when exception occurred - exception should be attributed to OS rather than P 

## 5. Configuration 
- Microkernel design - boot microkernel and only load other modules as needed
- When configuring the system, fix the specific set of devices
- By compile time fix the processor architecture 
- Configuration happens before compilation (during build) - produces conf.c and conf.h after compile - device switch table initialized 
- 3 sections to config - 
device type declaration section what driver functions 
% 
device specification section where on hardware 
% 
other configuration constants NPROC etc 
- Dynamic configuration - hardware uses single interrupt vector for the USB host controller 

## 6. File System 
- Control sharing on multiuser system
- Permissions are only checked when a file is opened 
- Each partition has an independent file system 
- Typed files/untyped files (a file is a sequence of bytes)

- Static allocation = space allocated before the file is used, dynamic = files grow as needed (has potential for starvation)
- Disk hardware cannot perform partial-block transfers
- Xinu file system - 
    - File system treats the directory as an array of pairs (file name, first index block for the file) = ldentry 
    - Directory contains struct ld_entry lfd_files
- Lfbyte = gives address of the next byte to read from the in-memory buffer 
- Xinu’s local file system prohibits concurrent access
    - – Only one active open can exist on a given file at a given time 

UNIX 
- OS maintains an open file table
- Each process has a file descriptor table
- A file descriptor is meaningless outside the process 
- Fork - reference count in the open file table is incremented 
- Inodes and data blocks 
- 13 pointers in inode
- Each directory contains a set of triples (type, file name, i-node number)
- Root directory is always at inode 2

## 7. Remote File System 
2 components - 
Server - 
    - Runs on a computer that has a LFS
Client - 
    - Part of an OS (sends req to server)
- Remote File paradigm - multiple clients can access server
- Solution - server serializes all incoming requests 
- Only one request can be outstanding at a time
- The upper half function forms and request message and calls rfscomm to send request to the server 
- Xinu RFS runs on Unix system
 - each open file has a mutual exclusion semaphore to prevent other processes from using the file while an operation proceeds
- The Xinu remote file access mechanism uses a synchronous approach that requires a message exchange for each operation, which means the remote file system access code does not need a separate process

## 8. File names and syntactic namespace 
- Safety - safe to use early binding to reveal how identifiers are mapped to internal values 
- Set of all valid file names 
- Gluing together file systems 
    - Create new root directory that is above the two file systems eg /F1/path_1 and /F2/path_2
- Prefix table - treat names as character strings

## 9. Remote access and a remote disk driver 
- Disk hardware can only do fetch and store 
- Allow OS to fetch and store blocks to a remote disk 
- Xinu remote disk driver software - upper hand read and write functions 
- Unlike conventional driver use a dedicated high priority communication process in place of a lower-half
- ￼<img width="455" height="281" alt="upper-half functions" src="https://github.com/user-attachments/assets/76bf73e9-c6b7-4724-97d3-252fd9a20757" />

- Virtualized disk for each client (provide client with illusion of separate physical disk)
- Caching - file systems exhibit temporal locality in which a given block is accessed repeatedly 
- 2 data structures 
    - A cache of recently-accessed disk blocks
    - Set of pending read or write requests 
* Invariant: at any time, a copy of block K in the cache contains the latest data written to block K
- Last write semantics maintained with queue (add requests to tail of queue)
- Using a semaphore to access the request queue does not guarantee strict temporal order of requests - because multiple processes may pass the semaphore if multiple spaces available
    - Solution - serialization mechanism 
- When the request queue gets empty, the communication process suspends itself so when you insert an entry into serial queue, make sure to resume it and suspend itself 
- Race condition - communication process runs at high priority
Solution - rdsars - does 2 things 
	- resumes communication process
	- suspends the calling process 
- Read and write on disk use length as the disk block number eg - read(RDISK, &buffer, 5)
- Request queue item = disk block number, operation, pointer to a buffer, id of proc waiting for the request to be fulfilled 
- htonl = local host byte order to network byte order 
    - Optimization of lower half software search the request queue backward to find last pending write request for the block 
- Request queue supports 3 operations - read, write, sync (Blocks the caller until all previously-written blocks have been stored on disk) it is handled locally 
- To agree on message formats use same header file but use a response bit that tells whether client or server 
- control(RDISK, RD_CTL_SYNC, 0); - provide control functions for remote disk - ensures all pending write operations have been completed before continuing 
- “A block number is not duplicated in both the request queue and cache”
* The cache only holds blocks that have been confirmed written to the remote server

## 10. DMA Devices and DMA Device Drivers
- DMA = allows device hardware to communicate with memory 
- DMA hardware = extracts data from memory and sends it 
- ￼<img width="604" height="275" alt="Illustration Of Buffers In A Ring" src="https://github.com/user-attachments/assets/30fb0ccc-f3ee-44dc-8fc2-938fd2784d07" />

- DMA hardware either sends data and marks the buffer empty or fills and buffer and makes it full. Moves to the next buffer in the ring 
- Best for block transfer 
- Optimizing - keep device busy 
- Ethernet = local DMA device, sits on my computer transfers network packets 

## 11. Networking and Protocol implementation 
- A computer can use local network communication or internet communication 
- ￼<img width="459" height="464" alt="Application Layer" src="https://github.com/user-attachments/assets/db9858f2-ec3c-40d8-8ade-72826d11dee2" />

- IP = internet protocol 
- UDP = user datagram protocol (defines protocol port numbers used to identify individual apps on a given computer)
- ARP = address resolution protocol (find ethernet address given IP address)
- DHCP = dynamic host configuration protocol (to obtain IP address)
- ICMP = internet control message protocol (ping program)
- MAC = media access control 
￼<img width="739" height="263" alt="Pasted Graphic 4" src="https://github.com/user-attachments/assets/242cb6d6-b9ab-41d6-8b10-488ce8015903" />

In this version of Xinu, an application can either 
– Use UDP to exchange messages with another application running on a computer on the Internet 
– Use ICMP to send a ping packet and receive a reply from an arbitrary computer on the Internet 
- ARP and DHCP merely provide support 
- Interface to communicate over the internet - Network code in the operating system responds by allocating an internal data structure, placing the information in the data structure, and returning a small integer descriptor that the application uses for communication
- Timeout and retransmission 
- Obtaining time of day from NTP server
- How can a computer send a DHCP message before the computer has an IP address? Use an all 0-s IP address and the sender’s address 
- Problem 
    - To use DHCP, network processes must be running (but are not started until late in the bootstrap sequence) 
    - Solution - delay using DHCP until app needs to use internet 
* Network code requires independent processes to ensure that an incoming packet is handled, even if no application is waiting for the packet SOLUTION = network input process
* The network input process must never call a function that blocks to wait for a reply
￼<img width="713" height="366" alt="• Netin handles incoming UDP and ARP packets" src="https://github.com/user-attachments/assets/e3caac4e-b42b-48d3-a4da-69496b49f709" />

netin process: Reads all incoming packets, NEVER blocks
ipout process: Sends outgoing IP packets, CAN block
This prevents deadlock!


DHCP - give me an IP address on this network 
ARP - used to acquire MAC address
ICMP - 

## 12. Device Management Device Drivers Device-Independent I / O
￼<img width="670" height="406" alt="Illustration Of A Device Switch Table" src="https://github.com/user-attachments/assets/ce398712-ddac-4119-961c-e5c77c7a7d3b" />


## 13. TTY driver 
- Console device displays output in a text window accepts input from user’s keyboard = character-oriented 
- Input = characters that come from keyboard
- Output = chars sent to screen 
- Hardware = uart 
- Buffer used in device driver 
- Console device interrupts when done sending a character 
- ￼<img width="670" height="406" alt="Illustration Of A Device Switch Table" src="https://github.com/user-attachments/assets/1945817e-c77d-4fdc-b39a-635aaaa5b36c" />

- Character output 
    - Output semaphore counts spaces in the device driver buffer
    - When you give a character to 
    - Modes - 
        - Raw = sends and receives individual bytes w no processing
        - Cooked = echoes input chars (backspace/erase)
        - Cbreak = handles some of the cooked mode functions
    - Input interrupt - one/more characters have arrived the driver must drain all chars from the device 
    - Kicking a device causes device to interrupt (if hardware is idle, kicking forces an interrupt)
