# Blob
A utility for embedding Lua scripts into C programs as pre-compiled bytecode.

Inspired by the myriad similar tools such as [bin2c], [lua2c], [squish], etc, this tool grew out of a personal need due to my tendency to:
* create projects that use an embedded Lua interpreter,
* create projects that use modified Lua interpreters,
* split even the most trivial projects across multiple Lua files.

Blob was an attempt to solve these problems all at once by allowing me to grab all of my scripts, along with my Lua intepreter of choice and simply mash it all into a single native binary.

Not useful for everyone, not suitable for all (or even most) situations, but when used appropriately it can simplify distribution immensely.

[bin2c]:http://lua-users.org/wiki/BinToCee
[lua2c]:http://wp.vpalos.com/669/lua2c
[squish]:http://code.matthewwild.co.uk/squish

## Usage
To install, simply copy `blob` to your shell search path of choice.

To run, simply type `blob` followed by the desired output file and one or more input files.
```
$ blob OUTPUT INPUTS...
```

### Standalone Mode (Default)
By default, Blob will include a `main` routine in the C source it generates. This `main` will create a new Lua state, load the embedded bytecode blob and then attempt to call into the global Lua function `main` which it expects to have been defined as part of the bytecode it loaded.

The source produced by this mode is a complete C program and can be compiled without modification.

### Embedded Mode
If the `--no-main` option is specified Blob will not provide a C `main` routine. It is expected that the resulting source file will be compiled as part of a larger C program.

### Multiple Files
If Blob is provided multiple input files it will combine them all into a single output file, however this is not always convenient. Sometimes it is desirable to generate a corresponding output file for each input. In this case it is still possible to have Blob generate a `main` routine in a separate file by using the `--main-only` option.

### B.Y.O. Lua Interpreter
When you run Blob it will run against whatever version of Lua is available at your shell. This will cause issues if your project uses a different version of Lua (be it newer, older, modified, etc) as they are not bytecode-compatible. To solve this, use the `--lua` option to specify your own Lua interpreter.

```
$ blob --lua=myproject/lua-5.3.2-modified/lua test.c test.lua
```

## Examples
Let's take an example Lua script:
```lua
function main()
  print("Hello world.")
end
```

### Standalone Mode
If we run the test script though Blob with the default options:
```
$ blob test.c test.lua
```
It will produce the following output:
```c
/**
 * test.c
 * Bytecode Blob precompiled for Lua 5.2
 *
 * This file was auto-generated by Blob as part of
 * the build process. It should not be modified or
 * committed to revision control, and will probably
 * be removed by a 'make clean'.
**/

#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdbool.h>

/**
 * luaopen_test()
 * test.lua
**/
bool luaopen_test(lua_State* L)
{
    const unsigned char blob[] = {
         27,  76, 117,  97,  82,   0,   1,   4,   8,   4,   8,   0,  25, 147,  13,  10,
         26,  10,   0,   0,   0,   0,   0,   0,   0,   0,   0,   1,   2,   3,   0,   0,
          0,  37,   0,   0,   0,   8,   0,   0, 128,  31,   0, 128,   0,   1,   0,   0,
          0,   4,   5,   0,   0,   0,   0,   0,   0,   0, 109,  97, 105, 110,   0,   1,
          0,   0,   0,   1,   0,   0,   0,   3,   0,   0,   0,   0,   0,   2,   4,   0,
          0,   0,   6,   0,  64,   0,  65,  64,   0,   0,  29,  64,   0,   1,  31,   0,
        128,   0,   2,   0,   0,   0,   4,   6,   0,   0,   0,   0,   0,   0,   0, 112,
        114, 105, 110, 116,   0,   4,  13,   0,   0,   0,   0,   0,   0,   0,  72, 101,
        108, 108, 111,  32, 119, 111, 114, 108, 100,  46,   0,   0,   0,   0,   0,   1,
          0,   0,   0,   0,   0,  10,   0,   0,   0,   0,   0,   0,   0,  64, 116, 101,
        115, 116,  46, 108, 117,  97,   0,   4,   0,   0,   0,   2,   0,   0,   0,   2,
          0,   0,   0,   2,   0,   0,   0,   3,   0,   0,   0,   0,   0,   0,   0,   1,
          0,   0,   0,   5,   0,   0,   0,   0,   0,   0,   0,  95,  69,  78,  86,   0,
          1,   0,   0,   0,   1,   0,  10,   0,   0,   0,   0,   0,   0,   0,  64, 116,
        101, 115, 116,  46, 108, 117,  97,   0,   3,   0,   0,   0,   3,   0,   0,   0,
          1,   0,   0,   0,   3,   0,   0,   0,   0,   0,   0,   0,   1,   0,   0,   0,
          5,   0,   0,   0,   0,   0,   0,   0,  95,  69,  78,  86,   0,
    };

    return !luaL_loadbuffer(L, (const char*)blob, sizeof(blob), "test.lua") && !lua_pcall(L, 0, 0, 0);
}

int main(int argc, char** argv)
{
    lua_State* L = luaL_newstate();
    luaL_openlibs(L);

    bool good = luaopen_test(L);

    if (!good) {
        fprintf(stderr, "%s\n", lua_tostring(L, -1));
        return 1;
    }

    lua_getglobal(L, "main");
    if (lua_type(L, -1) != LUA_TFUNCTION) {
        fprintf(stderr, "No 'main' function, aborting.\n");
        return 1;
    }

    int a;
    for (a = 0; a < argc; a++) {
        lua_pushstring(L, argv[a]);
    }

    if (lua_pcall(L, argc, 1, 0)) {
        fprintf(stderr, "%s\n", lua_tostring(L, -1));
        return 1;
    }

    lua_close(L);
    return 0;
}
```
Which can be compiled and run with no additional modification:
```
$ gcc -o test test.c -llua -ldl -lm
$ ./test
Hello world.
```

### Embedded Mode
Alternatively, if we run the test script though Blob with the `--no-main` option:
```
$ blob --no-main test.c test.lua
```
It will produce the following output:
```c
/**
 * test.c
 * Bytecode Blob precompiled for Lua 5.2
 *
 * This file was auto-generated by Blob as part of
 * the build process. It should not be modified or
 * committed to revision control, and will probably
 * be removed by a 'make clean'.
**/

#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdbool.h>

/**
 * luaopen_test()
 * test.lua
**/
bool luaopen_test(lua_State* L)
{
    const unsigned char blob[] = {
         27,  76, 117,  97,  82,   0,   1,   4,   8,   4,   8,   0,  25, 147,  13,  10,
         26,  10,   0,   0,   0,   0,   0,   0,   0,   0,   0,   1,   2,   3,   0,   0,
          0,  37,   0,   0,   0,   8,   0,   0, 128,  31,   0, 128,   0,   1,   0,   0,
          0,   4,   5,   0,   0,   0,   0,   0,   0,   0, 109,  97, 105, 110,   0,   1,
          0,   0,   0,   1,   0,   0,   0,   3,   0,   0,   0,   0,   0,   2,   4,   0,
          0,   0,   6,   0,  64,   0,  65,  64,   0,   0,  29,  64,   0,   1,  31,   0,
        128,   0,   2,   0,   0,   0,   4,   6,   0,   0,   0,   0,   0,   0,   0, 112,
        114, 105, 110, 116,   0,   4,  13,   0,   0,   0,   0,   0,   0,   0,  72, 101,
        108, 108, 111,  32, 119, 111, 114, 108, 100,  46,   0,   0,   0,   0,   0,   1,
          0,   0,   0,   0,   0,  10,   0,   0,   0,   0,   0,   0,   0,  64, 116, 101,
        115, 116,  46, 108, 117,  97,   0,   4,   0,   0,   0,   2,   0,   0,   0,   2,
          0,   0,   0,   2,   0,   0,   0,   3,   0,   0,   0,   0,   0,   0,   0,   1,
          0,   0,   0,   5,   0,   0,   0,   0,   0,   0,   0,  95,  69,  78,  86,   0,
          1,   0,   0,   0,   1,   0,  10,   0,   0,   0,   0,   0,   0,   0,  64, 116,
        101, 115, 116,  46, 108, 117,  97,   0,   3,   0,   0,   0,   3,   0,   0,   0,
          1,   0,   0,   0,   3,   0,   0,   0,   0,   0,   0,   0,   1,   0,   0,   0,
          5,   0,   0,   0,   0,   0,   0,   0,  95,  69,  78,  86,   0,
    };

    return !luaL_loadbuffer(L, (const char*)blob, sizeof(blob), "test.lua") && !lua_pcall(L, 0, 0, 0);
}
```
Which can now be included as part of a larger C project.

### Multiple Files
If we have multiple Lua scripts we can lump them together:
```
$ blob test.c test1.lua test2.lua test3.lua
```
Producing the following output:
```c
/**
 * test.c
 * Bytecode Blob precompiled for Lua 5.2
 *
 * This file was auto-generated by Blob as part of
 * the build process. It should not be modified or
 * committed to revision control, and will probably
 * be removed by a 'make clean'.
**/

#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdbool.h>

/**
 * luaopen_test1()
 * test1.lua
**/
bool luaopen_test1(lua_State* L)
{
    ...
}

/**
 * luaopen_test2()
 * test2.lua
**/
bool luaopen_test2(lua_State* L)
{
    ...
}

/**
 * luaopen_test3()
 * test3.lua
**/
bool luaopen_test3(lua_State* L)
{
    ...
}

int main(int argc, char** argv)
{
    ...

    bool good = luaopen_test1(L)
           && = luaopen_test2(L)
           && = luaopen_test3(L);

    ...
}
```

**OR** we can process them separately:
```
$ blob --no-main test1.c test1.lua
$ blob --no-main test2.c test2.lua
$ blob --no-main test3.c test3.lua
```
And then, if desired, generate a separate file:
```
$ blob --main-only test.c test1.lua test2.lua test3.lua
```
Containing _only_ a `main` routine:
```c
/**
 * test.c
 * Bytecode Blob precompiled for Lua 5.2
 *
 * This file was auto-generated by Blob as part of
 * the build process. It should not be modified or
 * committed to revision control, and will probably
 * be removed by a 'make clean'.
**/

#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdbool.h>

int main(int argc, char** argv)
{
    ...

    bool good = luaopen_test1(L)
           && = luaopen_test2(L)
           && = luaopen_test3(L);

    ...
}
```

## Compatibility
Blob requires Lua 5.1 or newer.

It was written against Lua 5.3, but I've tested it (briefly) against 5.2.4 and 5.1.5, both of which seemed to like it just fine.

## Releases
### 1.0 (2016-06-29)
Initial release; most likely the only release.

## License
Blob is distributed under the terms of the MIT license, which is reproduced within the Blob script.

