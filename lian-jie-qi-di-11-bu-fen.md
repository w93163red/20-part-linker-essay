# 链接器第11部分

## 归档

归档是一种传统的Unix打包格式。它们由ar程序创建，通常以.a扩展名命名。归档通过-l选项传递给Unix链接器。

虽然ar程序能够从任何类型的文件创建归档，但它通常用于将目标文件放入归档中。当它用于这个目的时，它会为归档创建一个符号表。符号表列出了任何目标文件在归档中定义的所有符号，并且对于每个符号指出了哪个目标文件定义它。最初，符号表是由ranlib程序创建的，但现在它总是由ar默认创建（尽管如此，许多Makefile仍然不必要地运行ranlib）。

当链接器看到一个归档时，它会查看归档的符号表。对于每个符号，链接器检查它是否看到了对该符号的未定义引用而没有看到定义。如果是这种情况，它就从归档中提取目标文件并将其包含在链接中。换句话说，链接器提取定义了已引用但尚未定义的符号的所有目标文件。

这个操作重复进行，直到无法从归档中定义更多符号。这允许归档中的目标文件引用同一归档中其他目标文件定义的符号，而不用担心它们出现的顺序。

注意，链接器考虑归档在命令行上相对于其他目标文件和归档的位置。如果一个目标文件在命令行上出现在归档之后，那么该归档将不会用于定义该目标文件引用的符号。

通常，如果归档为公共符号提供定义，链接器不会包含它们。你会记得，如果链接器看到一个公共符号后跟一个同名的定义符号，它会将公共符号视为未定义引用。只有在有其他原因将定义符号包含在链接中时，这才会发生；定义符号不会从归档中被拉出。

在旧的基于a.out的SunOS系统上，归档中的公共符号有一个有趣的转折。如果链接器看到一个公共符号，然后在归档中看到一个公共符号，它不会包含归档中的目标文件，但如果归档中的大小更大，它会将公共符号的大小更改为归档中的大小。C库在实现stdin变量时依赖于这种行为。

我的下一篇文章应该在周一发布。

