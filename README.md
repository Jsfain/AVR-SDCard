# SD Card Module
Used for executing the SPI mode-specific SD card command set using an AVR host.


# Purpose
To establish a set of functions for SD card access and control in SPI mode from an AVR microcontroller. The module is intended for standalone raw data access, but may be used in other situations, e.g. a physical disk layer for a file system.


# Disclaimers
* This is just a project I did for fun during quaratine. It has been tested against SDSC and SDHC type micro SD cards, and I expect full-sized cards to work as well, but no guarantee of functionality is made. I may or may not update in the future, though any feedback or suggestions are welcome.
* This module has only been tested against an ATmega1280 target, though it is expected it would be easily portable to other AVR targets with modification of SPI and USART ports, assuming sufficient memory.


# Warning
Use at your own risk. It is possible to write and erase data on an SD card using this module, so I suggest backing up any data you do not want to lose before using.


# Details
TARGET: ATmega1280 
LANGUAGE: C
The [AVR Toolchain](https://github.com/osx-cross/homebrew-avr) (AVR-GCC, AVR-LIBC, AVRDUDE) are used to compile, build, and ultimately download the project to the microcontroller. See the link above for specifics about this.


# SD Module Files
This SD card module includes the following source / header files

1) SD_SPI_BASE.C(H) - Basic functions required for interaction with the SD card. This includes an SD card initialization routine,  as well as functions for sending SD commands and get its response, and some error printing functions.
2) SD_SPI_DATA_ACCESS.C(H) - Some specialized functions for data access: READ/WRITE/ERASE. Requires SD_SPI_BASE
3) SD_SPI_MISC.C(H) - Some miscellaneous functions. Requires SD_SPI_BASE and SD_SPI_DATA_ACCESS.


# Additional Required Files
The following files are also required by the SD Card module and so are included in this repository. They technically are not considered part of this module itself, but rather are more general AVR functional modules taken from the avr-general repository. 

1) SPI.C(H) - required to interface with the AVR's SPI port used to transmit/receive data packets to/from the SD card.
2) USART.C(H) - required to interface with the AVR's USART port used to print messages and data to the screen.
3) PRINTS.H(C) - required to print integers (decimal, hex, binary) and strings to the screen via the USART.
    

# Who can use
Anyone. Use it. Modify it for specific purposes/systems, and if you want you can let me know if you found it helpful.


# Implementation
See the How To guide for specifc instructions on implementation.
This section will describes how to implement the module:

SD_SPI_BASE

Function List (descriptions are further down):
1) uint32_t  SD_InitializeSPImode(void)
2) void      SD_SendByteSPI(uint8_t data)
3) uint8_t   SD_ReceiveByteSPI(void)
4) void      SD_SendCommand(uint8_t cmd, uint32_t arg)
5) uint8_t   SD_GetR1(void) 
6) void      SD_PrintR1(uint8_t R1)
7) void      SD_PrintInitError(uint32_t err)


Definitions / Flags and default values:

#define HCS 0   
    * Host Capacity Support  
    * 0 for SD Standard Capacity (SDSC) cards. Currently only option supported.
    * 1 for SD High Capacity (SDHC) or Extended Capacity (SDXC). Not currently supported.
              
#define DATA_BLOCK_LEN 512 
    * Specifies length of data block. 
    * Should always be set to 512.

#define CS_LOW    SPI_PORT &= ~(1<<SS);  
    * Assert SD card's CS pin (pulls SS pin of SPI port low).  Use this to signal SD card prepare for interaction. 

#define CS_HIGH   SPI_PORT |= (1<<SS);
    * De-Assert SD card's CS pin, (pulls SS pin of the SPI port high).  Use to signal to SD card that communication has completed. 


List of Available SD card SPI-specific commands

GO_IDLE_STATE                 
SEND_OP_COND               
SWITCH_FUNC                  
SEND_IF_COND                 
SEND_CSD                         
SEND_CID                    
STOP_TRANSMISSION
SEND_STATUS              
SET_BLOCKLEN           
READ_SINGLE_BLOCK
READ_MULTIPLE_BLOCK
WRITE_BLOCK                 
WRITE_MULTIPLE_BLOCK
PROGRAM_CSD                
SET_WRITE_PROT             
CLR_WRITE_PROT             
SEND_WRITE_PROT          
ERASE_WR_BLK_START_ADDR 
ERASE_WR_BLK_END_ADDR     
ERASE                       
LOCK_UNLOCK        
APP_CMD                 
GEN_CMD                
READ_OCR              
CRC_ON_OFF          

//Application Specific Commands. Call APP_CMD immediately before calling any of these.
SD_STATUS               
SEND_NUM_WR_BLOCKS 
SET_WR_BLK_ERASE_COUNT
SD_SEND_OP_COND         
SET_CLR_CARD_DETECT  
SEND_SCR



R1 Error Response List:

OUT_OF_IDLE
IN_IDLE_STATE
ERASE_RESET
ILLEGAL_COMMAND
COM_CRC_ERROR
ERASE_SEQUENCE_ERROR
ADDRESS_ERROR
PARAMETER_ERROR
R1_TIMEOUT

An R1 response is the first byte returned by the SD card in response to any command sent by the host. The bits that are set in the R1 response byte determine which, if any, errors were encountered when sending the command as well as the current state of the SD card (In / Out of Idle state).  OUT_OF_IDLE and R1_TIMEOUT are not part of the SD card SPI mode standard, but will be set and used buy this implementation.  A state of OUT_OF_IDLE (R1 response = 0) indicates that the initialization routine has completed and that no errors were encountered in sending any commands.  While the card is being initialized the IN_IDLE_STATE bit must be returned by the SD card, along with any errors encountered / indicated in an R1 response.  The SD card SPI mode standard uses bits 0 to 6, bit 7 is reserved and should always be set to zero by the SD card. Bit 8 in this implementation is used to indicate a timeout (R1_TIMEOUT) if the SD card takes too long to return an R1 response. 

Use SD_GetR1(void) immediately after sending a command / argument to the SD card to get the R1 response. 
Pass the response to SD_PrintR1(R1) to print the returned response to the screen. 

example: send command/argument, get R1 and print R1.

SD_SendCommand(GO_IDLE_STATE, 0);
uint8_t r1 = SD_GetR1();
SD_PrintR1(r1)


Initialization Error Responses:
FAILED_GO_IDLE_STATE
FAILED_SEND_IF_COND
UNSUPPORTED_CARD_TYPE
FAILED_CRC_ON_OFF
FAILED_APP_CMD
FAILED_SD_SEND_OP_COND
OUT_OF_IDLE_TIMEOUT
FAILED_READ_OCR
POWER_UP_NOT_COMPLETE
FAILED_SET_BLOCKLEN
FAILED_SEND_CSD

These responses are returned by the intialization routine, SD_InitializeSPImode(), along with the most recently returned R1 response.  The initialization error responses occupy bits 8 through 19 of the response returned by SD_InitializeSPImode(), bits 0 to 7 are the most recent R1 response returned by the SD card during the intialization.

Use SD_PrintInitError( uint32_t initError ) to print the initialization error response.  This will not print the R1 portion of the response. To do that pass the bits 0 to 7 to SD_PrintR1( (uin8_t) initError )



// Card Types
#define SDHC 1 // or SDXC
#define SDSC 0 // standard capacity

typedef struct CardTypeVersion {
    uint8_t version;
    uint8_t type;
} CardTypeVersion;



example:
uint32_t resp = SD_InitializeSPImode(void);
SD_PrintR1( (uint8_t)resp );
SD_PrintInitError(resp);


FUNCTIONS
******************************************


uint32_t SD_InitializeSPImode(CardTypeVersion *ctv)

DESCRIPTION: 
Initializes an SD card into SPI mode and establishes the card type and version in *ctv.  The SD card is successfully initialized if INIT_SUCCESS and OUT_OF_IDLE are the initialization error and R1 portions of the initialization reponse, respectively.

The Initialization routine will also set the Type and Version in the CardTypeVersion struct passed in as the argument, which will be required for certain functions.  Once these are set they should not be changed.

ARGUMENTS:
pointer to CardTypeVersion struct.

RETURN:
uint32_t initialization response. This response is separated into two portions.  The lowest byte of the response is the most recent R1 response received from the SD card, with bits 8 to 19 is the initialization error response.

Use SD_PrintInitError(err) to print the initialization error response returned, and SD_PrintR1( (uint8_t) err) to print the R1 portion of the response.

EXAMPLE:
uint32_t resp = SD_InitializeSPImode(CardTypeVersion *cvt);
SD_PrintR1( (uint8_t)resp );
SD_PrintInitError(resp);


******************************************


void SD_SendByteSPI(uint8_t byte);

DESCRIPTION:
Send a single byte to the SD card via SPI.  This function, along with SD_ReceiveByteSPI() are the SPI interfaceing functions. All other functions interacting with the SD card must call this function to send a byte to the SD card.

ARGUMENTS:
uint8_t byte to send to the SD card.

RETURNS:
none

NOTES:
This function should be called as many times as necessary when sending a command or data to the SD card.  For instance SD_SendCommand() must call this function 6 times to completely send the a 48-bit command/argument to the SD card and writing to a 512 byte data block requires calling the function 64 times.


******************************************


void SD_SendCommand(uint8_t cmd, uint32_t arg);

DESCRIPTION:
Sends SD command, argument, and its CRC7 via SPI.

ARGUMENT:
    1) uint8_t command (cmd) from the SD card SPI command list.
    2) uint32_t argument (arg) to be sent as the argument for the command.

RETURNS:
none

NOTES:
This function calls pvt_CRC7 to calculate the CRC7 value to send with the command / argument pair.


******************************************


uint8_t SD_ReceiveByteSPI(void) 

DESCRIPTION:
Get the next byte returned by SD card via SPI. This function, along with SD_SendByteSPI(uint8_t) are the SPI interfaceing functions. All other functions interacting with the SD card must call this function to receive a byte from the SD card.

ARGUMENT:
none

RETURN:
uint8_t byte returned by the SD card.


NOTES:
    1) If a multi-byte value is expected to be returned then this function should be called at least that many times.  For example, reading in a 512 byte block requires calling this function 64 times.
    2) This function will return whatever byte value is present in the SPDR when the SPIF flag is set in SPI_MasterRead(). It is up to the calling function to determine if the returned value is valid.


******************************************


uint8_t SD_GetR1(void);

DESCRIPTION:
Gets the R1 response returned by an SD card for a given command / argument.  Call this function immediately after calling SD_SendCommand() to get the R1 response.  This function will call the SD_ReceiveByteSPI() to get the R1 byte, and then perform some additional handling.

ARGUMENT:
none

RETURNS:
uint8_t byte corresponding to the R1 response byte.

NOTES:
The R1 response is the first resopnse byte returned by an SD card in response to any command, therefore, this function should not be called at any other time except to get the first byte response, otherwise the returned value will not be the R1 response of the SD card. 


******************************************


void SD_PrintR1(uint8_t r1);

DESCRIPTION:
Prints the R1 response.

ARGUMENT:
uint8_t byte.  This byte should be the R1 response, otherwise the printed value is meaningless.

RETURN:
none

NOTES:
The R1 response is a set of flags occupying bits 0 to 6 of the returned byte. Bit 7 is reserved and should be 0 when returned by the SD card, however, SD_GetR1() will set this bit to R1_TIMEOUT, indicating the SD card did not return a response in an acceptable amount of time. The OUT_OF_IDLE state is used to indicate no errors and the card is not in the idle state (i.e. it has been successfully initialized).  During initialization, the card should be in the idle state and no error means only the IN_IDLE_STATE flag is set.


******************************************


void SD_PrintInitError(uint32_t err);

DESCRIPTION:
Prints the initialization error response flags returned by SD_InitializeSPImode().  This function does not print the R1 response portion of the initialization response.

ARGUMENT:
uint32_t initialization response returned by the response of SD_InitializeSPImode()

RETURNS:
none

NOTES:
The SD_InitializeSPImode() response includes the most recent R1 response. The R1 portion of the response will not be printed by this function and should be passed to SD_PrintR1() instead.


# Requirements
[AVR Toolchain](https://github.com/osx-cross/homebrew-avr)

# Reference Documents:
1) Atmel ATmega640/V-1280/V-1281/V-2560/V-2561/V Datasheet
2) SD Specifications Part 1: Physical Layer Specification - Simplified Specification Version 7.10
