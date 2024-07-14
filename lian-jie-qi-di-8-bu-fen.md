# 链接器第8部分

## ELF段

早些时候我说过可执行文件格式通常与目标文件格式相同。这对ELF来说是正确的，但有一个转折。在ELF中，目标文件由节组成：文件中的所有数据都是通过节表访问的。可执行文件和共享库通常包含一个节表，被nm这样的程序使用。但是操作系统和动态链接器不使用节表。相反，它们使用段表，它提供了文件的另一种视图。

ELF可执行文件或共享库中要加载到内存中的所有内容都包含在一个段内（目标文件没有段）。一个段有一个类型、一些标志、一个文件偏移量、一个虚拟地址、一个物理地址、一个文件大小、一个内存大小和一个对齐。文件偏移量指向一组连续的字节，这些字节是段的内容，即要加载到内存中的字节。当操作系统或动态链接器加载一个文件时，它会通过遍历段并将它们加载到内存中来完成（通常使用mmap系统调用）。动态链接器需要的所有信息 - 动态重定位、动态符号表等 - 都是通过存储在特殊段中的信息访问的。

虽然ELF可执行文件或共享库严格来说不需要任何节，但它们通常都有。可加载节的内容将完全落在单个段内。程序链接器从输入目标文件读取节。它对输出文件中的节进行排序和连接。它将所有可加载的节映射到输出文件的段中。它在输出文件段中布局节内容，尊重对齐和访问要求，以便段可以直接映射到内存中。节根据访问要求映射到段：通常所有只读节映射到一个段，所有可写节映射到另一个段。后者段的地址将被设置为在内存中的单独页面上开始，允许mmap对映射的页面设置不同的权限。

段标志是定义访问要求的位掩码。定义的标志是PF\_R、PF\_W和PF\_X，分别表示内容必须可读、可写或可执行。

段虚拟地址是运行时加载段内容的内存地址。物理地址官方上是未定义的，但在使用不使用虚拟内存的系统时通常用作加载地址。文件大小是文件中内容的大小。当段包含未初始化的数据时，内存大小可能大于文件大小；额外的字节将用零填充。段的对齐主要是信息性的，因为地址已经指定。

ELF段类型如下：

* PT\_NULL：段表中的空条目，被忽略。
* PT\_LOAD：段表中的可加载条目。操作系统或动态链接器加载所有这种类型的段。所有其他有内容的段的内容将完全包含在PT\_LOAD段内。
* PT\_DYNAMIC：动态段。这指向一系列动态标签，动态链接器使用这些标签来找到动态符号表、动态重定位和它需要的其他信息。
* PT\_INTERP：解释器段。这出现在可执行文件中。操作系统使用它来找到要为可执行文件运行的动态链接器的名称。通常所有可执行文件都有相同的解释器名称，但在某些操作系统上在不同的模拟模式下使用不同的解释器。
* PT\_NOTE：注释段。这包含操作系统或动态链接器可能使用的系统相关的注释信息。在GNU/Linux系统上，共享库通常有一个ABI标签注释，可用于指定共享库所需的最低内核版本。动态链接器在选择不同的共享库时使用这个。
* PT\_SHLIB：据我所知这是不使用的。
* PT\_PHDR：这表示段表的地址和大小。这在实践中不太有用，因为你必须已经找到段表才能找到这个段。
* PT\_TLS：TLS段。这保存TLS变量的初始值。
* PT\_GNU\_EH\_FRAME (0x6474e550)：GNU扩展，用于保存排序的展开信息表。这个表由GNU程序链接器构建。gcc的支持库使用它来快速找到异常的适当处理程序，而不需要在程序启动时注册异常帧。
* PT\_GNU\_STACK (0x6474e551)：GNU扩展，用于指示栈是否应该可执行。这个段没有内容。动态链接器将内存中的栈权限设置为这个段的权限。
* PT\_GNU\_RELRO (0x6474e552)：GNU扩展，告诉动态链接器在应用动态重定位后将给定的地址和大小设置为只读。这用于需要动态重定位的const变量。

### ELF节

现在我们已经讨论了段，让我们快速看一下ELF节的细节。ELF节比段更复杂，因为有更多类型的节。每个ELF目标文件和大多数ELF可执行文件和共享库都有一个节表。表中的第一个条目，节0，总是一个空节。

ELF节有几个字段：

* 名称
* 类型。我在下面讨论节类型。
* 标志。我在下面讨论节标志。
* 地址。这是节的地址。在目标文件中，这通常是零。在可执行文件或共享库中，它是虚拟地址。由于可执行文件通常通过段访问，这基本上是文档。
* 文件偏移量。这是文件中内容的偏移量。
* 大小。节的大小。
* 链接。根据节类型，这可能保存节表中另一个节的索引。
* 信息。这个字段的含义取决于节类型。
* 地址对齐。这是节所需的对齐。程序链接器在内存中布局节时使用这个。
* 条目大小。对于保存数据数组的节，这是一个数据元素的大小。

这些是程序链接器可能看到的ELF节类型：

* SHT\_NULL：空节。可以忽略此类型的节。
* SHT\_PROGBITS：保存程序位的节。这是一个带有内容的普通节。
* SHT\_SYMTAB：符号表。这个节实际上保存符号表本身。节内容是ELF符号结构的数组。
* SHT\_STRTAB：字符串表。这种类型的节保存以null结尾的字符串。这种类型的节用于符号的名称和节本身的名称。
* SHT\_RELA：重定位表。链接字段保存这些重定位适用的节的索引。这些重定位包括加数。
* SHT\_HASH：动态链接器用来加速符号查找的哈希表。
* SHT\_DYNAMIC：动态链接器使用的动态标签。通常PT\_DYNAMIC段和SHT\_DYNAMIC节将指向相同的内容。
* SHT\_NOTE：注释节。这以系统相关的方式使用。可加载的SHT\_NOTE节将成为PT\_NOTE段。
* SHT\_NOBITS：占用内存空间但没有关联内容的节。这用于零初始化的数据。
* SHT\_REL：重定位表，像SHT\_RELA但重定位没有加数。
* SHT\_SHLIB：据我所知这是不使用的。
* SHT\_DYNSYM：动态符号表。通常DT\_SYMTAB动态标签将指向与这个节相同的内容（尽管我还没有讨论动态标签）。
* SHT\_INIT\_ARRAY：这个节保存一个函数地址表，每个函数应该在程序启动时调用，或者对于共享库，当库被dlopen打开时调用。
* SHT\_FINI\_ARRAY：像SHT\_INIT\_ARRAY，但在程序退出时或dlclose时调用。
* SHT\_PREINIT\_ARRAY：像SHT\_INIT\_ARRAY，但在任何共享库初始化之前调用。通常共享库初始化器在可执行文件初始化器之前运行。这种节类型只能链接到可执行文件中，不能链接到共享库中。
* SHT\_GROUP：这用于将相关的节组合在一起，以便程序链接器可以在适当时作为一个单元丢弃它们。这种类型的节只能出现在目标文件中。这种类型的节的内容是一个标志字后跟一系列节索引。
* SHT\_SYMTAB\_SHNDX：ELF符号表条目只为节索引提供16位字段。对于有超过65536个节的文件，创建这种类型的节。它为每个符号保存一个32位字。如果符号的节索引是SHN\_XINDEX，可以通过查看SHT\_SYMTAB\_SHNDX节找到真实的节索引。
* SHT\_GNU\_LIBLIST (0x6ffffff7)：GNU扩展，预链接器用来保存预链接器找到的库列表。
* SHT\_GNU\_verdef (0x6ffffffd)：Sun和GNU扩展，用于保存版本定义（我稍后会讨论符号版本）。
* SHT\_GNU\_verneed (0x6ffffffe)：Sun和GNU扩展，用于保存从其他共享库需要的版本。
* SHT\_GNU\_versym (0x6fffffff)：Sun和GNU扩展，用于保存每个符号的版本。

这些是节标志的类型：

* SHF\_WRITE：节包含可写数据。
* SHF\_ALLOC：节包含应该是加载的程序映像一部分的数据。例如，这通常会为SHT\_PROGBITS节设置，而不为SHT\_SYMTAB节设置。
* SHF\_EXECINSTR：节包含可执行指令。
* SHF\_MERGE：节包含程序链接器可以合并以节省空间的常量。编译器可以对地址不重要的只读数据使用这种类型的节。
* SHF\_STRINGS：与SHF\_MERGE结合，这意味着该节保存可以合并的以null结尾的字符串常量。
* SHF\_INFO\_LINK：这个标志表示节中的info字段保存一个节索引。
* SHF\_LINK\_ORDER：这个标志告诉程序链接器，当它组合节时，这个节必须与link字段中的节保持相同的相对顺序。这可以用来确保地址表按预期顺序构建。
* SHF\_OS\_NONCONFORMING：如果程序链接器看到带有这个标志的节，并且不理解类型或所有其他标志，那么它必须发出错误。
* SHF\_GROUP：这个节出现在一个组中（见上面的SHT\_GROUP）。
* SHF\_TLS：这个节保存TLS数据。