# SminiFilter
This is a minifilter driver customized to protect against several filesystem operations. This minifilter comes included with a UM program that launches the minifilter driver (in case its not protecting
the system already) and communicates with the driver using IOCTLs to get the general and special logs of the filesystem by the driver

# Supported protection mechanisms:
- IRP_MJ_READ:
  1) replacing all text information in read buffer after a certain sequence (for example: "The password is:") with '*'s to hide private information
  2) disclosing all information read from a file in a certain parent directory (here i replaced all the information in the buffer with "ACCESS_DENIED XXX")
  3) encryption of all information in any file with a certain file suffix (here: ".... dirty.txt") with AES support in kernel mode library bcrypt.lib
- IRP_MJ_DIRECTORY_CONTROL:
  1) protection against seeing files inside a disclosed parent directory (manipulation of list returned by NtQueryDirectoryFile)
- IRP_MJ_SET_INFORMATION:
  1) protection against deletion of files inside a disclosed parent directory with IRP_MJ_SET_INFORMATION as the deletion primitive
  2) backup system of any undisclosed deleted files, saves the last version of the file before the deletion (currently only for specific paths)
  3) protection against copying a file into a disclosed path (with FileRenameInformation/Ex)
  4) protection against rename of a file to a certain file name (also FileRenameInformation/Ex)
  5) logging of different file changes with different operation flags
- IRP_MJ_CREATE:
  1) protection against deletion of files inside a disclosed parent directory with IRP_MJ_CREATE as the deletion primitive
  2) backup system of any undisclosed deleted files, saves the last version of the file before the deletion (currently only for specific paths)
  3) protection against getting a handle to a file/folder inside a disclosed directory
 - IRP_MJ_CLEANUP:
  1) regular tracing of calls to cleanup an object provided (usually a handle closed with CloseHandle/NtClose)
 - IRP_MJ_FILE_SYSTEM_CONTROL:
  1) Tracing calls to DeviceIoControl/KM similar functions, logging of passing method (BUFFERED/DIRECT/NEITHER) and log of I/O buffers and lengths
 - IRP_MJ_WRITE:
  1) Preventing writing information to specific root directories/files/directories
  2) Prevention of written vulnurable plaintext information by detecting known prefix patterns and writing only the information before these patterns
  3) encryption of all written information to specific files (also used AES here)
  4) if the write is valid, backup of the file before the last write operation is performed (currently only for specific paths)

  # Logging system of the driver and transfer to the UM program:
  1) the driver always saves a struct at the beginning of the allocated non-paged memory named MINIFILTER_STARTINFO. this struct includes basic
     information counters that were logged by the driver like read/write/create preop/postop operations that passed through the driver and the amount of
     detected special events (special events = all of the protection capabilities i described above, except the backup system of file changes/deletions)
  2) after this basic struct, the driver allocates more non-paged memory for each special event entry that was logged by the driver (if any occured). a special
     event entry includes a basic struct of information like the timestamp, information type and calling process (called DETECTED_ENTRY) + special event strings
     like the file name, file extension, parent directory, etc. these special entries provide extensive logging against special events on the file system in addition
     to actually handling them and providing extra protection
  3) this logging system adds more and more logs and entries as the time goes by until a certain IOCTL is sent to the driver by the UM program i provided
     (INFOPASS_IOCTL, 0x40002000), the IOCTL handler i implemented in the driver expects the PID of the process (first 8 bytes of parameters) and a dummy address
     for custom memory allocation for the database in the address range of the UM program (second 8 bytes of parameters, 16 bytes in total). the driver allocates
     enough memory for the logging list inside the calling process, copies the list from KM-UM memory and provides the base address of allocation and size of
     allocation in the first 16 bytes of the IOCTL output buffer. after transfering the logging list resets (counters are set to zero, special entries are
     deleted) and the logging list keeps getting filled until the next IOCTL
  4) the UM program gets a handle to the driver using its symbolic link and asks for an update of the logging list in 30 second intervals. the UM programs
     goes over the whole logging list and parses the saved information on the output window to show the logged events to the user, then it frees the memory and
     waits for the next update

  # TODO list:
  1) Implement more capabilities i still did not think of

  # Installation process:
  1) extract the project files
  2) install the ShminiFilterInstall.inf file
  3) move the ShminiFilter.sys driver into System32\drivers
  4) run ShminiClient.exe with administrative priviledges
