
# XCode Application Building. From source code to binary.

## SHORT HAND

- **Compile Sources** Компилируются некоторые файлы (ваш код), а скомпилированный код копируется в приложение. Эта фаза сборки обычно применяется ко всем целевым файлам . т; это файлы исходного кода, образующие цель. В настоящее время цель содержит файлы ViewController .m, AppDelegate. m и main. m. Если вы добавите в свой проект новый класс, то должны будете указать, что он является частью целевого приложения, а файл . ш будет автоматически добавлен в фазу Compile Sources.

- **Link Binary With Libraries** Некоторые библиотеки (обычно каркасы) связываются со скомпилированным кодом (теперь он является бинарным), так что приложение теперь будет ожидать, что во время его выполнения эти библиотеки будут находиться на устройстве.

- **Copy Bundle Resources** Некоторые файлы копируются в приложение, чтобы ваш код или система могли найти их во время выполнения приложения. Например, если приложение имеет пиктограмму, ее нужно скопировать в приложение, чтобы устройство могло его найти и отобразить на экране. Эта фаза сборки в настоящее время применяется к каталогу ресурсов; в каталог изображения можно добавить любые изображения, которые будут скопированы в ваше приложение. В настоящий момент каталог содержит файлы InfoPlist. strings и . storyboard. Копирование не обязательно означает создание идентичной копии. Файлы некоторых типов при копировании в комплект приложения автоматически обрабатываются специальным образом. Например, копирование каталога ресурсов означает, что пиктограммы и заставки в каталоге записываются на верхний уровень комплекта приложения; копирование файла . storyboard означает, что он будет преобразован в файл . storyboardc, который в свою очередь представляет собой комплект, состоящий из nib-файлов.

- **Run Scripts** Custom scripts can be added to the build process. (e.g. Fabric disrtibution tool)

## Detailed 

Roughly speaking, the compiler has two tasks: converting our Objective-C code into low-level code, and analyzing our code to make sure we didn’t make any obvious mistakes.

These days, Xcode ships with clang as the compiler. Wherever we write compiler, you can read it as clang. clang is the tool that takes Objective-C code, analyzes it, and transforms it into a more low-level representation that resembles assembly code: LLVM Intermediate Representation. LLVM IR is low level, and operating system independent. LLVM takes instructions and compiles them into native bytecode for the target platform. This can either be done just-in-time or at the same time as compilation.

LLVM that takes care of translating the IR code into native bytecode for different platforms (Intel and ARM).

**When compiling a source file, the compiler proceeds through several phases.**
```
% clang -ccc-print-phases hello.m

0: input, "hello.m", objective-c
1: preprocessor, {0}, objective-c-cpp-output
2: compiler, {1}, assembler
3: assembler, {2}, object
4: linker, {3}, image
5: bind-arch, "x86_64", {4}, image
```

### STEP 1. Preprocessing

The preprocessor handles a macro processing language, which means it will replace macros in your text by their definitions.
```objc
#import <Foundation/Foundation.h> // can use '@class MyClass;' in .h file to reduce unused code in header file, and add import to the implementation file
```
The preprocessor will take that line, and substitute it with the contents of that file. If that header file contains any other macro definitions, they will get substituted too.
This is the reason why people tell you to keep your header files free of imports as much as you can, because anytime you import something, the compiler has to do more work.

#### 1.1 Custom Macros
```objc
#define MY_CONSTANT 
#define MY_MACRO(x) x
```

Now, anytime you write MY_CONSTANT after this line, it’ll get replaced by 4 before the rest of compilation starts.

#### 1.2 Tokenization (Lexing)
After preprocessing is done, every source .m file now has a bunch of definitions. This text is converted from a string to a stream of tokens.
```objc
int main() {
  NSLog(@"hello, %@", @"world");
  return 0;
}
```

```
int 'int'        [StartOfLine]  Loc=<hello.m:4:1>
identifier 'main'        [LeadingSpace] Loc=<hello.m:4:5>
l_paren '('             Loc=<hello.m:4:9>
r_paren ')'             Loc=<hello.m:4:10>
l_brace '{'      [LeadingSpace] Loc=<hello.m:4:12>
identifier 'NSLog'       [StartOfLine] [LeadingSpace]   Loc=<hello.m:5:3>
l_paren '('             Loc=<hello.m:5:8>
at '@'          Loc=<hello.m:5:9>
string_literal '"hello, %@"'            Loc=<hello.m:5:10>
comma ','               Loc=<hello.m:5:21>
at '@'   [LeadingSpace] Loc=<hello.m:5:23>
string_literal '"world"'                Loc=<hello.m:5:24>
r_paren ')'             Loc=<hello.m:5:31>
semi ';'                Loc=<hello.m:5:32>
return 'return'  [StartOfLine] [LeadingSpace]   Loc=<hello.m:6:3>
numeric_constant '0'     [LeadingSpace] Loc=<hello.m:6:10>
semi ';'                Loc=<hello.m:6:11>
r_brace '}'      [StartOfLine]  Loc=<hello.m:7:1>
eof ''          Loc=<hello.m:7:2>
```
We can see that each token consists of a piece of text and a source location. The source location is from before macro expansion, so that if something goes wrong, clang can point you to the right spot.

#### 1.3 Parsing
On this step stream of tokens is parsed into an abstract syntax tree. After parsing, a program is now available as an abstract syntax tree: a tree that represents the original program.

```objc
#import <Foundation/Foundation.h>

@interface World
- (void)hello;
@end

@implementation World
- (void)hello {
  NSLog(@"hello, world");
}
@end

int main() {
   World* world = [World new];
   [world hello];
}
```

```
@interface World- (void) hello;
@end
@implementation World
- (void) hello (CompoundStmt 0x10372ded0 <hello.m:8:15, line:10:1>
  (CallExpr 0x10372dea0 <line:9:3, col:24> 'void'
    (ImplicitCastExpr 0x10372de88 <col:3> 'void (*)(NSString *, ...)' <FunctionToPointerDecay>
      (DeclRefExpr 0x10372ddd8 <col:3> 'void (NSString *, ...)' Function 0x1023510d0 'NSLog' 'void (NSString *, ...)'))
    (ObjCStringLiteral 0x10372de38 <col:9, col:10> 'NSString *'
      (StringLiteral 0x10372de00 <col:10> 'char [13]' lvalue "hello, world"))))


@end
int main() (CompoundStmt 0x10372e118 <hello.m:13:12, line:16:1>
  (DeclStmt 0x10372e090 <line:14:4, col:30>
    0x10372dfe0 "World *world =
      (ImplicitCastExpr 0x10372e078 <col:19, col:29> 'World *' <BitCast>
        (ObjCMessageExpr 0x10372e048 <col:19, col:29> 'id':'id' selector=new class='World'))")
  (ObjCMessageExpr 0x10372e0e8 <line:15:4, col:16> 'void' selector=hello
    (ImplicitCastExpr 0x10372e0d0 <col:5> 'World *' <LValueToRValue>
      (DeclRefExpr 0x10372e0a8 <col:5> 'World *' lvalue Var 0x10372dfe0 'world' 'World *'))))
```
Every node in the abstract syntax tree is annotated with the original source position, so that if there’s any problem later on, clang can warn about your program and give you the correct location.

#### 1.4 Static Analysis

- Static Type Checking
- IVars Use Checking
- Object init Checking
- ARC Leaks Checking
- (ELSE ON clang -> lib/StaticAnalyzer/Checkers)

### STEP 2. Compiling

Now, once your code is fully tokenized, parsed, and analyzed by clang, it can generate the LLVM code for you.

**C** Hello.c
```objc
#include <stdio.h>

int main() {
  printf("hello world\n");
  return 0;
}
```

**LLVM Intermediate Code** Hello.ll
```
; ModuleID = 'hello.c'
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.9.0"

@str = private unnamed_addr constant [12 x i8] c"hello world\00"

; Function Attrs: nounwind ssp uwtable
define i32 @main() #0 {
  %puts = tail call i32 @puts(i8* getelementptr inbounds ([12 x i8]* @str, i64 0, i64 0))
  ret i32 0
}

; Function Attrs: nounwind
declare i32 @puts(i8* nocapture readonly) #1

attributes #0 = { nounwind ssp uwtable "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "stack-protector-buffer-size"="8" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { nounwind }

!llvm.ident = !{!0}

!0 = metadata !{metadata !"Apple LLVM version 6.0 (clang-600.0.41.2) (based on LLVM 3.5svn)"}
```

**OBJECTIVE C** Hello.c
```objc
#include <stdio.h>
#import <Foundation/Foundation.h>

int main() {
  NSLog(@"%@", [@5 description]);
  return 0;
}
```

**LLVM Intermediate Code** Hello.ll
```
define i32 @main() #0 {
  %1 = load %struct._class_t** @"\01L_OBJC_CLASSLIST_REFERENCES_$_", align 8
  %2 = load i8** @"\01L_OBJC_SELECTOR_REFERENCES_", align 8, !invariant.load !5
  %3 = bitcast %struct._class_t* %1 to i8*
  %4 = tail call %0* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %0* (i8*, i8*, i32)*)(i8* %3, i8* %2, i32 5)
  %5 = load i8** @"\01L_OBJC_SELECTOR_REFERENCES_2", align 8, !invariant.load !5
  %6 = bitcast %0* %4 to i8*
  %7 = tail call %1* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %1* (i8*, i8*)*)(i8* %6, i8* %5)
  tail call void (i8*, ...)* @NSLog(i8* bitcast (%struct.NSConstantString* @_unnamed_cfstring_ to i8*), %1* %7)
  ret i32 0
}
```
The most important lines are line 4, which creates the NSNumber object, line 7, which sends the description message to the number object, and line 8, which logs the string returned from the description message.

#### 2.1 Optimizations
LLVM and clang can do some complex optimizations of your code.
To use optimizations you can pass ```-O3``` flag to clang.
You can define this in Project Build Settings

<img src="https://github.com/m4stodon/ios-guide/blob/master/Additional/Images/XCode_Series/Compiler_Optimization.png">

> Note: Usefull tools for mannual code analyses and code generation: libclang - C lib, ClangKit - Objc wrapper, LibTooling - C++ the most powerfull lib provided by clang itself.


## STEP 3. Assemblying
The assembler converts the (human-readable) assembly code into machine code. It creates a target object file, often simply called object file. These files have a .o file ending. If you build your app with Xcode, you’ll find these object files inside the Objects-normal folder inside the derived data directory of your project.

### 3.1 Segments and Sections
An executable will have multiple sections, i.e. parts. Different parts of the executable will each go into their own section, and each section will in turn go inside a segment.
```
% xcrun size -x -l -m a.out 
Segment __PAGEZERO: 0x100000000 (vmaddr 0x0 fileoff 0)
Segment __TEXT: 0x1000 (vmaddr 0x100000000 fileoff 0)
    Section __text: 0x37 (addr 0x100000f30 offset 3888)
    Section __stubs: 0x6 (addr 0x100000f68 offset 3944)
    Section __stub_helper: 0x1a (addr 0x100000f70 offset 3952)
    Section __cstring: 0xe (addr 0x100000f8a offset 3978)
    Section __unwind_info: 0x48 (addr 0x100000f98 offset 3992)
    Section __eh_frame: 0x18 (addr 0x100000fe0 offset 4064)
    total 0xc5
Segment __DATA: 0x1000 (vmaddr 0x100001000 fileoff 4096)
    Section __nl_symbol_ptr: 0x10 (addr 0x100001000 offset 4096)
    Section __la_symbol_ptr: 0x8 (addr 0x100001010 offset 4112)
    total 0x18
Segment __LINKEDIT: 0x1000 (vmaddr 0x100002000 fileoff 8192)
total 0x100003000
```

Our ```a.out``` file has four segments. Some of these have sections.
When we run an executable, the VM (virtual memory) system maps the segments into the address space (i.e. into memory) of the process. Mapping is very different in nature, but if you’re unfamiliar with the VM system, simply assume that the VM loads the entire executable into memory – even though that’s not what’s really happening. The VM pulls some tricks to avoid having to do so.
When the VM system does this mapping, segments and sections are mapped with different properties, namely different permissions.
The ```__TEXT segment``` contains our code to be run. It’s mapped as read-only and executable. The process is allowed to execute the code, but not to modify it. The code can not alter itself, and these mapped pages can therefore never become dirty.
The ```__DATA``` segment is mapped read/write but non-executable. It contains values that need to be updated.
The first segment is ```__PAGEZERO```. It’s 4GB large. Those 4GB are not actually in the file, but the file specifies that the first 4GB of the process’ address space will be mapped as non-executable, non-writable, non-readable. This is why you’ll get an ```EXC_BAD_ACCESS``` when reading from or writing to a NULL pointer, or some other value that’s (relatively) small. It’s the operating system trying to prevent you from causing havoc.
Within segments, there are sections. These contain distinct parts of the executable. In the ```__TEXT``` segment, the ```__text``` section contains the compiled machine code. ```__stubs``` and ```__stub_helper``` are used for the dynamic linker (dyld). This allows for lazily linking in dynamically linked code. ```__const``` (which we don’t have in our example) are constants, and similarly ```__cstring``` contains the literal string constants of the executable (quoted strings in source code).
The ```__DATA``` segment contains read/write data. In our case we only have ```__nl_symbol_ptr``` and ```__la_symbol_ptr```, which are non-lazy and lazy symbol pointers, respectively. The lazy symbol pointers are used for so-called undefined functions called by the executable, i.e. functions that are not within the executable itself. They’re lazily resolved. The non-lazy symbol pointers are resolved when the executable is loaded.
Other common sections in the ```__DATA``` segment are ```__const```, which will contain constant data which needs relocation. An example is ```char * const p = "foo";``` – the data pointed to by p is not constant. The ```__bss``` section contains uninitialized static variables such as static int a; – the ANSI C standard specifies that static variables must be set to zero. But they can be changed at run time. The ```__common``` section contains uninitialized external globals, similar to static variables. An example would be int a; outside a function block. Finally, ```__dyld``` is a placeholder section, used by the dynamic linker.

### 3.2 Section Content
```
% xcrun otool -s __TEXT __text a.out 
a.out:
(__TEXT,__text) section
0000000100000f30 55 48 89 e5 48 83 ec 20 48 8d 05 4b 00 00 00 c7 
0000000100000f40 45 fc 00 00 00 00 89 7d f8 48 89 75 f0 48 89 c7 
0000000100000f50 b0 00 e8 11 00 00 00 b9 00 00 00 00 89 45 ec 89 
0000000100000f60 c8 48 83 c4 20 5d c3 
```

This is the code of our app. Since -s ```__TEXT``` ```__text``` is very common, otool has a shortcut to it with the -t argument. We can even look at the disassembled code by adding -v:
```
% xcrun otool -v -t a.out
a.out:
(__TEXT,__text) section
_main:
0000000100000f30    pushq   %rbp
0000000100000f31    movq    %rsp, %rbp
0000000100000f34    subq    $0x20, %rsp
0000000100000f38    leaq    0x4b(%rip), %rax
0000000100000f3f    movl    $0x0, 0xfffffffffffffffc(%rbp)
0000000100000f46    movl    %edi, 0xfffffffffffffff8(%rbp)
0000000100000f49    movq    %rsi, 0xfffffffffffffff0(%rbp)
0000000100000f4d    movq    %rax, %rdi
0000000100000f50    movb    $0x0, %al
0000000100000f52    callq   0x100000f68
0000000100000f57    movl    $0x0, %ecx
0000000100000f5c    movl    %eax, 0xffffffffffffffec(%rbp)
0000000100000f5f    movl    %ecx, %eax
0000000100000f61    addq    $0x20, %rsp
0000000100000f65    popq    %rbp
0000000100000f66    ret
```

In a similar fashion, we can look at other sections:
```
% xcrun otool -v -s __TEXT __cstring a.out
a.out:
Contents of (__TEXT,__cstring) section
0x0000000100000f8a  Hello World!\n
```
OR
``` 
% xcrun otool -v -s __TEXT __eh_frame a.out 
a.out:
Contents of (__TEXT,__eh_frame) section
0000000100000fe0    14 00 00 00 00 00 00 00 01 7a 52 00 01 78 10 01 
0000000100000ff0    10 0c 07 08 90 01 00 00 
```

### 3.3 Arbitrary Sections 
You can add arbitrary data as a section to your executable with the -sectcreate linker flag. This is how you’d add a Info.plist to a single file executable. The Info.plist data needs to go into a ```__info_plist``` section of the ```__TEXT``` segment. 


## STEP 4. Linking
The linker will resolve symbols between object files and libraries.
```printf()``` is a function in the libc library. Somehow, the final executable needs to be able to know where in memory the ```printf()``` is, i.e. what the address of the ```_printf``` symbol is. The linker takes all object files (in our case, only one) and the libraries (in our case, implicitly libc) and resolves any unknown symbols (in our case, the ```_printf```). It then encodes into the final executable that this symbol can be found in libc, and the linker then outputs the final executable that can be run: ```a.out```.

### 4.1 Symbols and linking
Our small app was put together from two object files. The ```Foo.o``` object file contains the implementation of the ```Foo``` class, and the ```helloworld.o``` object file contains the ```main()``` function and calls/uses the ```Foo``` class.
Furthermore, both of these use the Foundation framework. The ```helloworld.o``` object file uses it for the autorelease pool, and it indirectly uses the Objective-C runtime in form of the ```libobjc.dylib```. It needs the runtime functions to make message calls. This is similar to the ```Foo.o``` object file.
All of these are represented as so-called symbols. We can think of a symbol as something that’ll be a pointer once the app is running, although its nature is slightly different.
Each function, global variable, class, etc. that is defined or used results in a symbol. When we link object files into an executable, the linker (ld(1)) resolves symbols as needed between object files and dynamic libraries.

```
% xcrun nm -nm helloworld.o
                 (undefined) external _OBJC_CLASS_$_Foo
0000000000000000 (__TEXT,__text) external _main
                 (undefined) external _objc_autoreleasePoolPop
                 (undefined) external _objc_autoreleasePoolPush
                 (undefined) external _objc_msgSend
                 (undefined) external _objc_msgSend_fixup
0000000000000088 (__TEXT,__objc_methname) non-external L_OBJC_METH_VAR_NAME_
000000000000008e (__TEXT,__objc_methname) non-external L_OBJC_METH_VAR_NAME_1
0000000000000093 (__TEXT,__objc_methname) non-external L_OBJC_METH_VAR_NAME_2
00000000000000a0 (__DATA,__objc_msgrefs) weak private external l_objc_msgSend_fixup_alloc
00000000000000e8 (__TEXT,__eh_frame) non-external EH_frame0
0000000000000100 (__TEXT,__eh_frame) external _main.eh
```

These are all symbols of that file. ```_OBJC_CLASS_$_Foo``` is the symbol as the Foo Objective-C class. It’s an undefined, external symbol of the Foo class. External means it’s not private to this object file, as opposed to non-external symbols which are private to the particular object file. Our helloworld.o object file references the class Foo, but it doesn’t implement it. Hence, its symbol table ends up having an entry marked as undefined.
Next, the ```_main``` symbol for the ```main()``` function is also external because it needs to be visible in order to get called. It, however, is implemented in ```helloworld.o``` as well, and resides at address 0 and needs to go into the ```__TEXT```,```__text``` section. Then there are four Objective-C runtime functions. These are also undefined and need to be resolved by the linker.
If we turn toward the Foo.o object file, we get this output:

```
% xcrun nm -nm Foo.o
0000000000000000 (__TEXT,__text) non-external -[Foo run]
                 (undefined) external _NSFullUserName
                 (undefined) external _NSLog
                 (undefined) external _OBJC_CLASS_$_NSObject
                 (undefined) external _OBJC_METACLASS_$_NSObject
                 (undefined) external ___CFConstantStringClassReference
                 (undefined) external __objc_empty_cache
                 (undefined) external __objc_empty_vtable
000000000000002f (__TEXT,__cstring) non-external l_.str
0000000000000060 (__TEXT,__objc_classname) non-external L_OBJC_CLASS_NAME_
0000000000000068 (__DATA,__objc_const) non-external l_OBJC_METACLASS_RO_$_Foo
00000000000000b0 (__DATA,__objc_const) non-external l_OBJC_$_INSTANCE_METHODS_Foo
00000000000000d0 (__DATA,__objc_const) non-external l_OBJC_CLASS_RO_$_Foo
0000000000000118 (__DATA,__objc_data) external _OBJC_METACLASS_$_Foo
0000000000000140 (__DATA,__objc_data) external _OBJC_CLASS_$_Foo
0000000000000168 (__TEXT,__objc_methname) non-external L_OBJC_METH_VAR_NAME_
000000000000016c (__TEXT,__objc_methtype) non-external L_OBJC_METH_VAR_TYPE_
00000000000001a8 (__TEXT,__eh_frame) non-external EH_frame0
00000000000001c0 (__TEXT,__eh_frame) non-external -[Foo run].eh
```

The fifth-to-last line shows that ```_OBJC_CLASS_$_Foo``` is defined and external to ```Foo.o``` – it has this class’s implementation.
Foo.o also has undefined symbols. First and foremost are the symbols for ```NSFullUserName()```, ```NSLog()```, and ```NSObject``` that we’re using.
When we link these two object files and the Foundation framework (which is a dynamic library), the linker tries to resolve all undefined symbols. It can resolve ```_OBJC_CLASS_$_Foo``` that way. For the others, it will need to use the Foundation framework.
When the linker resolves a symbol through a dynamic library (in our case, the Foundation framework), it will record inside the final linked image that the symbol will be resolved with that dynamic library. The linker records that the output file depends on that particular dynamic library, and what the path of it is. That’s what happens with the ```_NSFullUserName```, ```_NSLog```, ```_OBJC_CLASS_$_NSObject```, ```_objc_autoreleasePoolPop```, etc. symbols in our case.
We can look at the symbol table of the final executable a.out and see how the linker resolved all the symbols:

```
% xcrun nm -nm a.out 
                 (undefined) external _NSFullUserName (from Foundation)
                 (undefined) external _NSLog (from Foundation)
                 (undefined) external _OBJC_CLASS_$_NSObject (from CoreFoundation)
                 (undefined) external _OBJC_METACLASS_$_NSObject (from CoreFoundation)
                 (undefined) external ___CFConstantStringClassReference (from CoreFoundation)
                 (undefined) external __objc_empty_cache (from libobjc)
                 (undefined) external __objc_empty_vtable (from libobjc)
                 (undefined) external _objc_autoreleasePoolPop (from libobjc)
                 (undefined) external _objc_autoreleasePoolPush (from libobjc)
                 (undefined) external _objc_msgSend (from libobjc)
                 (undefined) external _objc_msgSend_fixup (from libobjc)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
0000000100000e50 (__TEXT,__text) external _main
0000000100000ed0 (__TEXT,__text) non-external -[Foo run]
0000000100001128 (__DATA,__objc_data) external _OBJC_METACLASS_$_Foo
0000000100001150 (__DATA,__objc_data) external _OBJC_CLASS_$_Foo
```

We see that all the Foundation and Objective-C runtime symbols are still undefined, but the symbol table now has information about how to resolve them, i.e. in which dynamic library they’re to be found.
The executable also knows where to find these libraries:

```
% xcrun otool -L a.out
a.out:
    /System/Library/Frameworks/Foundation.framework/Versions/C/Foundation (compatibility version 300.0.0, current version 1056.0.0)
    /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1197.1.1)
    /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 855.11.0)
    /usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
```

These undefined symbols are resolved by the dynamic linker dyld(1) at runtime. When we run the executable, dyld will make sure that ```_NSFullUserName```, etc. point to their implementation inside Foundation, etc.
We can run nm(1) against Foundation and check that these symbols are, in fact, defined there:

```
% xcrun nm -nm `xcrun --show-sdk-path`/System/Library/Frameworks/Foundation.framework/Foundation | grep NSFullUserName
0000000000007f3e (__TEXT,__text) external _NSFullUserName 
```

**The dyld’s Shared Cache**
And these in turn will use countless other frameworks and dynamic libraries. The list of all dynamic libraries that need to get loaded gets large quickly. And the list of interdependent symbols even more so.
In order to shortcut this process, the dynamic linker on OS X and iOS uses a shared cache that lives inside /var/db/dyld/. For each architecture, the OS has a single file that contains almost all dynamic libraries already linked together into a single file with their interdependent symbols resolved. When a Mach-O file (an executable or a library) is loaded, the dynamic linker will first check if it’s inside this shared cache image, and if so, use it from the shared cache. Each process has this dyld shared cache mapped into its address space already. This method dramatically improves launch time on OS X and iOS.



# Resources

- [PewPewTheSpells.com Build System](https://pewpewthespells.com/blog/xcode_build_system.html)
- [objc.io Build Process](https://www.objc.io/issues/6-build-tools/build-process/)
- [objc.io Mach-O](https://www.objc.io/issues/6-build-tools/mach-o-executables/)
- [objc.io The Compiler](https://www.objc.io/issues/6-build-tools/compiler/)









































