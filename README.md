# luajit-c-embedding

Here are some notes/instructions for embedding the LuaJIT in C. Initially, we follow along with [Marek Vavrusa's blog post](https://en.blog.nic.cz/2015/08/12/embedding-luajit-in-30-minutes-or-so/). Later, we move beyond and see how to interact with things in Torch.

### Prerequisites

Before we begin, you'll need to install [LuaJIT](http://luajit.org/install.html) and [Luarocks](https://github.com/keplerproject/luarocks/wiki/Download). If you use the Torch installation, you may have an issue with linking, so either download releases or build directly from source.

## Hello, World!

Let's start by creating a Lua script named hello.lua which contains ```print('Hello, World!')```. Next, create a C file hello-luajit.c which looks like this (modified from the above blog post):

https://gist.github.com/klgraham/6d4f937abf39e56d48381a3c40369920

To compile the C, use:

```gcc hello-luajit.c $(pkg-config --cflags --libs luajit) -lm -ldl -pagezero_size 10000 -image_base 100000000 -o hello.out```, noting that ```-pagezero_size 10000 -image_base 100000000``` only needs to be included on macOS systems.

You can now execute the Lua script with ```./hello.out hello.lua```.

### Understanding the C Code

To interact with Lua from C we will use [Lua's C API](http://www.lua.org/pil/24.1.html). Let's look at the each part of hello-luajit.c.

- ```luaL_newstate```: Starts up Lua and returns a new Lua state
- ```luaL_openlibs```: loads the Lua standard libraries
- ```luaL_loadfile```: Loads the Lua script
- ```lua_pcall```: runs the Lua script and returns an error code
- ```lua_close```: closes Lua

## Using the Lua Stack

Lua uses a stack to exchange values with C. To get a value from Lua, you call Lua to push the value onto the stack. To pass a value from C to Lua, you call Lua to push the value onto the stack and then pop the value. When pushing a value onto the stack, it's important to know if the stack has room for the new value. The Lua Programming Book has [a nice example of this](http://lua-users.org/wiki/SimpleLuaApiExample). Let's look at an example.

Suppose we want to compute the factorial of a number in Lua and make the result of the computation available in C. The Lua code is in the file factorial.lua:

```
function factorial(n)
  local result = 1
  for i = 1, n do
    result = result * i
  end
  return result
end
```

From C, we'll call factorial.lua and then get the result from the Lua stack and print to stdout.

https://gist.github.com/klgraham/35bd1d2b0f4b57be5d2bbc30f6721d46

This example is pretty simple:

1. First, we load the Lua file containing the factorial function
- Call ```lua_pcall``` to execute the script and create the functions
- Call ```lua_getglobal``` to specify which function you wish to call
- Push the function arguments to the stack, in order
- Call ```lua_pcall``` again, specifying the number of arguments and return values
- Use ```lua_tointeger``` to get the return value from the top of the stack

You'll notice that at the end we pop the returned value from the stack with ```lua_pop```. For such a simple example, this is not needed since we're killing Lua afterwards, but for more complex uses you'll definitely want to clean up the stack before moving on.

You can compile with ```gcc factorial-luajit.c $(pkg-config --cflags --libs luajit) -lm -ldl -pagezero_size 10000 -image_base 100000000  -o fact.out```.


