#PE/COFF Specification Addendum

## Deterministic PE/COFF File

PE/COFF file is _deterministic_ if it is produced by a tool that guarantees that the entire content of the file is based solely on documented inputs given to the tool (such as source files, resource files, compiler options, etc.) rather than ambient environment variables (such as the current time, the operating system, the bitness of the process running the tool, etc.). The content of the file can be reproduced exactly (bit for bit) given the same inputs.

The value of field TimeDateStamp in COFF File Header of a deterministic PE/COFF file does not indicate the date and time when the file was produced and should not be interpreted that way. Instead the value of the field is derived from a hash of the file content. The algorithm used to calculate this value is an implementation detail of the tool that produced the file.

## Debug Directory

Image files contain an optional debug directory that indicates what form of debug information is present and where it is. This directory consists of an array of debug directory entries whose location and size are indicated in the image optional header.

PE/COFF Specification defines the structure of Debug Directory in section 5.1.1. Each entry has a Type and a pointer to data. The format of the data is specific to the Type of the debug entry and not prescribed by the PE/COFF specification. The following paragraphs specify format of data of debug directory entries produced and consumed by CLI (Common Language Infrastructure) compilers, debuggers and other tools.

### CodeView Debug Directory Entry (type 2)

*Version Major=0, Minor=0* of the data format:

| Offset | Size | Field          | Description                                                    |
|:-------|:-----|:---------------|----------------------------------------------------------------|
| 0      | 4    | Signature      | 0x52 0x53 0x44 0x53 (ASCII string: "RSDS") |
| 4      | 16   | Guid           | GUID (Globally Unique Identifier) of the associated PDB.  
| 20     | 4    | Age            | Iteration of the PDB. The first iteration is 1. The iteration is incremented each time the PDB content is augmented.
| 24     |      | Path           | UTF-8 NUL-terminated path to the associated .pdb file |

Guid and Age are used to match PE/COFF image with the associated PDB. 

The associated .pdb file may not exist at the path indicated by Path field. If it doesn't the Path, Guid and Age can be used to find the corresponding PDB file locally or on a symbol server. The exact search algorithm used by tools to locate the PDB depends on the tool and its configuration.

If the containing PE/COFF file is deterministic the Guid field above and DateTimeStamp field of the directory entry are  calculated deterministically based solely on the content of the associated .pdb file. Otherwise the value of Guid is random and the value of DateTimeStamp indicates the time and date that the debug data was created.

*Version Major=any, Minor=0x504d* of the data format has the same structure as above. The Age shall be 1. The format of the associated .pdb file is Portable PDB. The Major version specified in the entry indicates the version of the Portable PDB format. Together 16B of the Guid concatenated with 4B of the TimeDateStamp field of the entry form a PDB ID that should be used to match the PE/COFF image with the associated PDB (instead of Guid and Age). Matching PDB ID is stored in the #Pdb stream of the .pdb file.

### Deterministic Debug Directory Entry (type 16)

The entry doesn't have any data associated with it. All fields of the entry, but Type shall be zero.

Presence of this entry indicates that the containing PE/COFF file is deterministic. 

### Embedded Portable PDB Debug Directory Entry (type 17)

Declares that debugging information is embedded in the PE file at location specified by PointerToRawData. 

*Version Major=any, Minor=0x0100* of the data format:

| Offset | Size           | Field            | Description                                           |
|:-------|:---------------|:-----------------|-------------------------------------------------------|
| 0      | 4              | Signature        | 0x4D 0x50 0x44 0x42                                   |
| 4      | 4              | UncompressedSize | The size of decompressed Portable PDB image           |
| 8      | SizeOfData - 8 | PortablePdbImage | Portable PDB image compressed using Deflate algorithm | 

If this entry is present and the reader recognizes this entry the debugging information for the PE file shall be read from the embedded data. If a CodeView entry is also present it shall be ignored. 

> Note: Including both entries enables a tool that does not recognize Embedded Portable PDB entry to locate debug infomration as long as it is also available in a file specified in CodeView entry. Such file can be created by extracting the embedded Portable PDB image to a separate file.

The Major version specified in the entry indicates the version of the Portable PDB format. The Minor version indicates the version of the Embedded Portable PDB data format.

The value of Stamp field in the entry shall be 0.