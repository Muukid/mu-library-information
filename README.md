# Introduction

The simplest definition of a 'mu' library is:

a low-overhead dual-public domain/MIT-licensed* C library whose source code and dependencies are stored entirely in one file.

Elaboration on the ' * ' for licensing is found in the license section.

Note that although the source code and dependencies are stored entirely in one file, it may use system-defined libraries that need to be linked to, which will be defined in the library's respective documentation.

It is up to the user to decide whether or not a mu library would be better to use in comparison to a traditionally structured project depending on the user's circumstances. Id est, use them if you like, don't if you don't.

# General structure

There are certain general structure choices made in mu libraries. They're not strict rules, but mu libraries almost always follow them, and can safely be assumed unless otherwise specified within the documentation for the respective library.

## Header/Implementation macros

mu libraries are split into two general parts within the single file: header and implementation.

The header provides what is essentially the interface for the library's usage, defining only the things that are necessary to use it. The header's contents are automatically defined upon inclusion of the file, using a macro (usually formatted like `MUX_H`) to check if it hasn't been defined yet and define it if so.

The implementation provides the actual source code that makes the library function (99% just actually providing the code for the defined functions). The implementation is triggered by a manual macro defined before inclusion of the file (usually formatted like `MUX_IMPLEMENTATION`), and, if this macro goes undefined, the actual source code for the library will never be provided and will most likely cause errors and/or warnings.

Noe that both sections provide their respective section for its included libraries (ie, if the library muX has a dependency on muY, the header for muY will be stored in the header section of muX and the implementation for muY will be stored in the implementation section of muX). If a library's header/implementation section is already defined, it will be skipped, but a check on a possible version mismatch will be performed and will throw a warning. This functionality is overridable by defining `MU_CHECK_VERSION_MISMATCHING`.

## C/C++ standards and general compiler compatability

mu libraries usually follow both the C99 and C++11 standards. They're tested for compilation with gcc, clang, g++, and clang++, with all warnings enabled and all warnings turned into errors, but tested rarely with MSVC due to its difficulty to use through the terminal and its inaccessibility (or impracticality to access) through non-Windows operating systems. Generally, though, the restrictions on the aforementioned compilers make the code compatible with MSVC.

Note that I hate Microsoft. (Except for you, Raymond Chen)

## Usual function format

The average function format in mu libraries goes like this:

```c
MUDEF RETURNTYPE mux_SUBJECT_THINGBEINGDONE(muxContext* context, muxResult* result, ...)
```

If the library is simple enough that it doesn't need a context, then result is usually the first parameter.

Oftentimes, there are macros for these functions that allow the user to not explicity pass context and/or result values, to which the function automatically assumes the global context and its respective stored result value; generally, these macros follow the format of:

```c
#define mu_SUBJECT_THINGBEINGDONE(...) mux_SUBJECT_THINGBEINGDONE(mux_global_context, &mux_global_context->result, __VA_ARGS__)
#define mu_SUBJECT_THINGSBEINGDONE_(result, ...) mux_SUBJECT_THINGBEINGDONE(mux_global_context, result, __VA_ARGS__)
```

This gives the user three ways to use the average function:

```c
// Fully explicit:
mux_SUBJECT_THINGBEINGDONE(&context, &result, ...);
// Explicit result-checking with assumed context:
mu_SUBJECT_THINGBEINGDONE_(&result, ...);
// Assumed context, result goes into context:
mu_SUBJECT_THINGBEINGDONE(...);
```

This is done because each function starting with the same parameters can be annoying, but assuming either the context or result can make the code unusable with multiple threads or contexts, so all three options are provided.

Some functions end up breaking this mold, usually because they don't need a context, they can't fail (so no need for result checking), or both, to which the macro that would usually use such such parameters goes undefined.

## Context structuring

The lower level and more simplistic mu libraries are usually don't need to store data across functions, meaning most functions can be called without need for prior setup via other parts of the library. However, higher level mu libraries usually need to store information, and the common structure for this is a context creation/destruction structure, in which a context is created & associated with a pointer to a context structure (usually at the beginning of the program's lifetime), and a destruction function is called on the context when it is no longer needed (usually right before the program exits). This structure usually looks like this:

```c
muxContext context;
mux_context_create(&context, MU_TRUE);

// ...mu_whatever(...)...
// context.result

mux_context_destroy(&context);
```

Inbetween this, the context created is set as the global context and is assumed as such by macros to functions that don't take in a context; the global context being set as such is triggered by the `MU_TRUE` in the context creation function for the example above. The result of functions is stored in `context.result`, with `result` being the library's respective result enumerator.

This model also allows for more explicit context definitions, allowing for multiple contexts to be defined:

```c
muxContext context1, context2;
mux_context_create(&context1, MU_FALSE);
mux_context_create(&context2, MU_FALSE);

// ...mux_whatever(&context1...)...
// ...mux_whatever(&context2...)...

mux_context_destroy(&context1);
mux_context_destroy(&context2);
```

### Warning about context pointers

The assumed context is valid as long as the pointer initially passed to the `mux_context_create` function is valid; the following code would cause undefined behaviour:

```c
muxContext init() {
	muxContext context;
	mux_contex_create(&context);
	return context;
}

int main() {
	// ...

	muxContext context = whatever(); // Context is created, but the pointer is only defined within the scope of the function, meaning that the value returned is likely meaningless.

	mu_whatever(...); // This will now cause undefined behavior, as the original pointer is now dangling.
}
```

The inner library uses the pointer initially passed to the context creation function, so it is advised to ensure that the pointer is valid as long as the context exists.

### Object referencing

Functions that rely on context structuring usually return types that are just macros for the `void*` type, acting as a reference to dynamically allocated memory to be dereferenced when used by the library, follwing a general syntax of:

```c
muObject object = mu_object_create(...); // 'object' is now a pointer.
...
...mu_object_do_something(...object...)... // The library is now using the pointer to hold information about the object.
...
object = mu_object_destroy(...object...); // The object's data is now wrapped up and destroyed, with the pointer being freed. Now 'object' is 0.
```

The reason why `object` needs to be dynamically allocated and freed is because the exact memory that `object` will take up cannot always be known beforehand. For example, in muCOSA, a library that can handle window management, exactly *what* a 'window' is defined as can depend on the window system currently running. It could be assumed that only one window system can be defined at a time, but then that makes operating systems that can use multiple window systems (for example, Linux with X11 and Wayland) need to have separately-compiled versions of muCOSA, stored rather as separate library files or as separate executables. A `union` could be used, but risks taking up more memory than is needed.

## Inclusion of muUtility

The library muUtility is usually included in a mu library. This library defines many of the essential things used by virtually all mu libraries, such as `size_m`, `(u)intN_m` types, `muBool`, `MUDEF`, `muByte`, et cetera. More explicit information about what muUtility provides is defined in its documentation, and if you have any questions about types and macros that you see repeated in all mu libraries, they're probably defined in muUtility and elaborated upon in muUtility's documentation.

### Usage of non-secure functions

mu libraries often use non-secure functions that will trigger warnings on certain compilers. These warnings are, to put it lightly, dumb, so the header section of muUtility defines `_CRT_SECURE_NO_WARNINGS` (unless `MU_SECURE_WARNINGS` is defined). However, it is not guaranteed that this definition will actually turn the warnings off, which at that point, they have to be manually turned off by the user, or the functions have to be overriden.

## Result enumerator

mu libraries usually provide a result enumerator that is used to represent how a function performed. This enumerator also usually includes the enumerators for the libraries it depends on, ie, if library muX depends on muY, muX's result enumerator list most likely looks like:

```
MUX_SUCCESS,
... (other muX enumerators)

MUX_MUY_SUCCESS,
... (other muY enumerators)
```

## Low overhead

Most mu libraries are low overhead, meaning that they aim to perform as few computations and checks as possible to achieve their tasks.

### Fail safety

The fail safety of mu libraries generally hinges on the following principle:

Proper error-checking will be performed on operations reasonably outside of the user's control, and error-checking will not be performed on factors in which the user has reasonable control over and can reasonably check for validity.

This principle establishes low overhead at the cost of safety; for example, virtually every function that takes in a context can crash the application instantly if passed an invalid pointer to a context (including null pointers).

### Thread safety

Most mu libraries are *not* thread-safe for low overhead purposes. Most mu libraries assume that each 'object' (a `void*` reference that needs a context to be created) will not be accessed by more than one thread at a time, so if a program is using a mu library in a multi-threaded manner, it is best practice to protect each object that will be used by multiple threads under a thread lock unless otherwise specified by the library.

## Character encoding

Most mu libraries use UTF-8 as their character encoding, meaning that when a string is specified by the user or returned by the library (besides the name functions), it can be safely assumed that the encoding is UTF-8 unless otherwise explicity noted in the documentation for the library if relevant.

## Name functions

mu libraries usually provide functions for converting enumerators that they provide from their value to a `const char*` string representation. This is done for the purpose of debugging and is most useful with function result enumerator checking.

These name functions are usually only defined if a previous macro is defined, usually following the format of `MUX_NAMES`.

## Major/Minor/Patch version macros

mu libraries usually define their version in 3 respective major/minor/patch macros within the header section, usually following the format of `MUX_VERSION_MAJOR`, `MUX_VERISON_MINOR`, and `MUX_VERSION_PATCH`.

## Overridable C standard library dependencies

mu libraries usually have every single dependency on the C standard library overridable. This includes everything defined by the respective included standard file and used at some point in the library, such as functions, types, and macros.

# Documentation

The documentation provided with a mu library (usually in the form of a single file, `README.md`) explicity states what the library does and how to use it, bringing up every relevant function, variable, macro, C standard library dependency, other libraries stored within the library, struct, enumerator, et cetera.

The documentation is usually also defined in the file itself with comments, using special symbols interpreted by `muDOG`, a custom-made tool for inline documentation for mu libraries that allows Markdown to be written within the file.

Note that the detail about the other libraries included within itself are not provided, only the fact that they're provided and the version.

# License

A license is provided with a given mu library (stored at the end of the file and also accompanied with a `license.md` file) that is dual, one being public domain and one being MIT, with the user deciding which license to use. The idea behind this is to have a license as useable and flexible as possible, but having it only as public domain can lead to legal issues that I'm not gonna pretend like I understand :D (I'm a programmer, not a lawyer).

Note that the public domain license also allows you to include any mu library in your software without changing its license.

## Note on OpenGL & Apache 2.0 exception

Some mu libraries have dependencies on OpenGL, meaning that they need to import OpenGL using some loader. No matter the loader, the code for this process is guaranteed to be based on the Khronos specification for OpenGL, which is licensed under Apache 2.0. This means that any mu library or component of it that imports OpenGL also has the Apache 2.0 license attached to it. This will be noted in the proper sections of the library that specify license information.

Note that Vulkan doesn't have this, as it has a built-in exception clause for its license that the mu libraries fall under.

[Altogether, however, you probably don't need to worry about it.](https://github.com/KhronosGroup/OpenGL-Registry/issues/376#issuecomment-596187053)

# Demos

mu libraries are usually provided with a folder for demos that provide example programs (with many explanation comments) that use the library in various demonstrative ways. This is provided in case the documentation isn't sufficient or you just wish to get the gist for how to use the library quickly.

# FAQ

## Why C?

C was chosen for the mu libraries due to its high compatibility with devices, libraries, and even other programming languages in many cases. C++ also has this, but to a significantly lower extent, especially in regards to compatability with other programming languages.

## Why all one file?

I'm not going to argue for or against a header-only single-file library format, as I don't think I'm personally experienced enough in the entire C/C++ library ecosystem and relevant build systems to make such a call with any degree of confidence.

What I will say is that when I'm programming in C or C++, I often find myself thinking one of two things when importing another library:

"Man, this is so easy because it's header-only single-file."

or

"Man, this sucks."

I'm sorry, but I'm a sucker for simplicity, and I've never really run into a case where I'm using a header-only single-file library and had a hard time importing or using it. For my cases, they work, and they work well, so I think that it's safe to presume that the same would apply for many others.

## How do you navigate these files?

Most of these files make use of tab characters to allow code editors to automatically open/close various relevant sections. The high levels of indentation caused by this make it only easily viewable if the relevant code editor has tab characters defined with only ~1-3 spaces.

I won't say that this makes it a piece of cake, but it definitely makes it usable in my experience.

# Usual inner file contents

The usual inner file contents of a mu file are meant to be relatively tiny (extreme emphasis on the 'relatively' part of that sentence), which means that it doesn't contain things such as demos (see the demos section) or many explanatory comments (that's the job of the documentation).

The following is the usual inner file contents of a mu file in order from top to bottom.

## Beginning explanation comment

The file begins with the following format:

```c
/*
FILENAME - AUTHOR
SHORT DESCRIPTION
No warranty implied; use at your own risk.

Licensed under MIT License or public domain, whichever you prefer.
More explicit license information at the end of file.

...
*/
```

This just gives a brief explanation of what the file is for anybody who opens it. The `...` could be virtually anything; it's usually notes such as `@MENTION ...` or `@TODO ...`.

## Inline documentation

From here on out, a lot of inline documentation later scanned by the tool `muDOG` to generate Markdown documentation is defined in comments.

## Header section

The next section is usually the header section of the library, being wrapped like so:

```c
#ifndef MUX_H
	#define MUX_H

	...
	
	#ifdef __cplusplus
	extern "C" { // }
	#endif
	
	...
	
	#ifdef __cplusplus
	}
	#endif
#endif /* MUX_H */
```

Within the first `...` is the header section of other libraries that this library depends on, and within the second `...` is the actual contents of the header.

### Inclusion of other library headers

Other libraries' header sections are included within this section. If they've already been defined, a check for version mismatching is performed that can be turned off with `MU_CHECK_VERSION_MISMATCHING` being defined beforehand. This is, generally, what that looks like:

```c
/* Library version n.n.n header */
	
	#if !defined(MU_CHECK_VERSION_MISMATCHING) && defined(LIBRARY_H) && \
		(LIBRARY_VERSION_MAJOR != n || LIBRARY_VERSION_MINOR != n || LIBRARY_VERSION_PATCH != n)
		
		#pragma message("[...] Library's header has already been defined, but version doesn't match the version that this library is built for. This may lead to errors, warnings, or unexpected behavior. Define MU_CHECK_VERSION_MISMATCHING before this to turn off this message.")

	#endif

	#ifndef LIBRARY_H
		...
	#endif /* LIBRARY_H */
```

### Version major/minor/patch definition

The major, minor, and patch macros are usually defined within the header section, following this general format:

```c
#define MUX_VERSION_MAJOR n
#define MUX_VERSION_MINOR n
#define MUX_VERSION_PATCH n
```

### C standard library dependencies

Any C standard library dependencies not defined beforehand are usually defined within the header section, following this general format:

```c
/* C standard library dependencies */
	
	#if !defined(mu_function) || \
		!defined(type_m)      || \
		!defined(MU_MACRO)
		
		#include <relevantfile.h>
		
		#ifndef mu_function
			#define mu_function function
		#endif

		#ifndef type_m
			#define type_m type
		#endif

		#ifndef MU_MACRO
			#define MU_MACRO MACRO
		#endif
	
	#endif
	
	...
	
```

Note that these C standard library dependencies are overridable if defined beforehand, as this formatting would suggest, and are all listed in the documentation for their respective mu library.

### Enumerators

Most mu libraries define their enumerators within the header section, the most important of which is usually a result enumerator to check for the result of a function, following this general formatting:

```c
/* Enums */

	MU_ENUM(muxResult,
		MUX_SUCCESS,
		...
	)

```

Note that this uses the `MU_ENUM` macro function provided by muUtility instead of the usual actual `enum` keyword in C because I've run into issues with MSVC and enumerators. Note that I hate Microsoft (of course, except for you, Raymond Chen).

### Functions

Most mu libraries define their functions within the header section. It usually follows this general formatting:

```c
/* Functions */

	/* Names */

		#ifdef MUX_NAMES
			...
		#endif
	
	/* Initiation / Termination */
		
		MUDEF void mux_init(muxResult* result);
		MUDEF void mux_term(muxResult* result);
	
	...

```

## Implementation section

The implementation section for most mu libraries comes after the header section, following this general formatting:

```c
#ifdef MUX_IMPLEMENTATION
	
	...

	#ifdef __cplusplus
	extern "C" { // }
	#endif

	...

	#ifdef __cplusplus
	}
	#endif

#endif /* MUX_IMPLEMENTATION */
```

Within the first `...` is the implementation section of other libraries that this library depends on, and within the second `...` is the actual contents of the implementation.

### Inclusion of other library implementations

Other libraries' implementation sections are included within this section. This is, generally, what that looks like:

```c
/* Library version n.n.n implementation */

	#ifndef LIBRARY_IMPLEMENTATION
		#define LIBRARY_IMPLEMENTATION
		
		#ifdef LIBRARY_IMPLEMENTATION
			...
		#endif /* LIBRARY_IMPLEMENTATION */
	#endif

```

## End of file license

Just like the beginning comment says, the file ends with the license format:

```c
/*
------------------------------------------------------------------------------
This software is available under 2 licenses -- choose whichever you prefer.
------------------------------------------------------------------------------
ALTERNATIVE A - MIT License
Copyright (c) YEAR AUTHOR
Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
------------------------------------------------------------------------------
ALTERNATIVE B - Public Domain (www.unlicense.org)
This is free and unencumbered software released into the public domain.
Anyone is free to copy, modify, publish, use, compile, sell, or distribute this
software, either in source code form or as a compiled binary, for any purpose,
commercial or non-commercial, and by any means.
In jurisdictions that recognize copyright laws, the author or authors of this
software dedicate any and all copyright interest in the software to the public
domain. We make this dedication for the benefit of the public at large and to
the detriment of our heirs and successors. We intend this dedication to be an
overt act of relinquishment in perpetuity of all present and future rights to
this software under copyright law.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN
ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
------------------------------------------------------------------------------
*/


```

Note that there are purposefully two empty lines at the end for compilers who throw warnings or errors for a file not ending in a newline character. There are two because sometimes copying and pasting removes the last line if it's a newline character, hopefully bypassing such a check with a second newline.
