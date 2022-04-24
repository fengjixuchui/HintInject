# HintInject
HintInject is a shellcode embedder and loader that I developed while playing with the PE file and Import Directory Table structures. It takes a raw shellcode file and puts the shellcode in chunks into the Hint/Name Table entries which are reachable via the Import Lookup Table of a fake imported DLL entry on the loader executable. The loader then merges these chunks to execute the shellcode

# DLL Loading Process
Before explaining how HintInject works, I want to briefly mention about the DLL Loading process by referencing a post in Stack Overflow.

To get the information of required DLLs and functions/imports, one should look at the `Import Directory Table` firstly. Import Directory Table is a table of entries, one entry for every imported DLL. These entries hold a pointer to the imported DLL name, a pointer to `Import Lookup Table`, a pointer to `Import Address Table`, and other fields for different information. 

Simply, Import Lookup Table leads to the information of imports, and Import Address Table leads to addresses of imports. However, when the executable is on the disk, or just before the DLL loading process, Import Address Table are identical to that of the Import Lookup Table. Import Address Table's content is overwritten with the address of the imports during the DLL Loading process. 

Without going into some detail, we can take a look at the diagram below to see the relationship between these three tables.

<p align="center">
<img src="https://user-images.githubusercontent.com/26549173/164817710-18070017-a8f1-4346-8a1f-2c275152c074.png">
</p>

If we go into a little more detail, the Import Lookup Table doesn't hold names of imports directly. For the functions that are imported by name, it holds RVAs of `Hint/Name Table` entries. These entries store the function names as Null terminated ASCII strings. Therefore, the struct definition of a Hint/Name Table entry is as follows:
```
typedef struct _IMAGE_IMPORT_BY_NAME {
    WORD    Hint;
    CHAR   Name[1];
} IMAGE_IMPORT_BY_NAME, *PIMAGE_IMPORT_BY_NAME; 
```
The Hint field here is actually an index into the export name pointer table of the DLL. It is used to accelerate finding the position of that import. Therefore, we can summarize the relationship between the Import Lookup Table and the Hint/Name table with the diagram below.

<p align="center">
<img src="https://user-images.githubusercontent.com/26549173/164833587-a9a39601-5a0f-49a4-b2ce-4b66c5b28d91.png">
</p>

To conclude, in order to find a DLL and its imports, one should follow these steps:

1. Go to Import Directory Table from the DataDirectory array of Optional Header.
2. Traverse the Import Directory Table entries for finding the desired DLL by checking its name field.
3. After finding the correct entry, go to its Import Lookup Table by using the OriginalFirstThunk field.
4. Traverse the all entries in the Import Lookup Table which are pointers to Hint/Name Table entries.
5. Check import names from the Name field of Hint/Name Table entries.

# How HintInject Works?
Recently, while examining the ImportDLLInjection technique shared by x86matthew, I saw how to add a fake entry to the Import Directory of DLLs loaded in memory. Afterwards, I wanted to develop a little project that adds a fake entry directly to the import table of a binary on disk, both for fun and to refresh my knowledge. While reviewing the DLL Loading process to develop this project, the Hint field in the Hint/Name Table entries used in this process caught my attention. 

According to the MSDN document, I learned that this field is used by Windows Loader to find the address of that import, which is imported by name, directly from the export name table of the DLL it is in. However, in the same document, it was stated that if the function cannot be found by using this field, that function will be searched via a binary search operation in the DLL's export name table. Based on this sentence, I thought that putting an incorrect value for this field would not disrupt the DLL Loading process.

As I mentioned above, there is an entry to the Hint/Name table corresponding to each function to be imported. In other words, we have 2 bytes to use for each import. By combining multiple imports, enough Hint fields can be obtained to store malicious shellcodes. As a result, one can use a loader binary to embed the shellcode in the Hint/Name entries of the fake import DLL entry. That loader binary can reach these entries to merge the shellcode to be executed during runtime.

HintInject can be used to create such a loader that holds the shellcode in its Hint/Name table. It firsts creates a new section named `.rrdata`, and copies the current import directory into this section. After that, it appends a new fake entry whose imports will be used to hold input shellcode. The remaining bytes of the section are used as to store Import Lookup Table, Import Address Table, DLL name and Hint/Name table of the new fake entry. As the last step, HintInject uses the Hint fields of imports to put chunks of input shellcode. Approximately, the memory layout of the new section is as follows:

<p align="center">
<img width="460" height="400" src="https://user-images.githubusercontent.com/26549173/164948439-a7e05320-a290-4a44-9dcf-3ac680b9c275.png">
</p>

# Files

# Usage

# Limitations

# References
- https://stackoverflow.com/questions/32841368/whats-the-difference-between-the-import-table-import-adress-table-and-import
- https://0xrick.github.io/win-internals/pe6/
- https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#hintname-table
- https://www.x86matthew.com/view_post?id=import_dll_injection
