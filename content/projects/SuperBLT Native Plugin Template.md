---
title: SuperBLT Plugin Template
description: My journey of porting the SuperBLT Plugin Template from C++ to Rust so I don't have to get cancer when writing the dll for Heister's Haptics
date: 2024-05-11
tags:
  - Payday2
  - Mod
  - Modding
  - HeistersHaptics
  - SuperBLT
  - Porting
---

Have you ever asked yourself what the best way to learn a language in depth is?
Yeah don't do this I'm kind of masochistic.

Anyway in here I will attempts to unfuck the [SuperBLT Native Plugin Template](https://gitlab.com/SuperBLT/native-plugin-template) and 
[SuperBLT Native Plugin Library](https://gitlab.com/SuperBLT/native-plugin-library) and rewrite them entirely in rust. This was mainly done for my other Project [[PD2 Heister's Haptics]].
However rust is far from being my best language, as is C++ so this is and will be a struggle.

## Initial Victories
The initial idea was to use [mlua](https://github.com/mlua-rs/mlua/tree/master) to handle all the Lua functionality and then just export the required symbols for SuperBLT to be able to interface with it.

However I can't read properly so I missed an export for the setup function. 
The error for that looks like this:

```
04:16:21 PM FATAL ERROR:  (C:\projects\payday2-superblt\src\InitiateState.cpp:320) mods/HeistersHaptics/mod.lua:11: Invalid dlhandle - missing setup_state func!
```

Afterwards I simply ran [dumpbin](https://learn.microsoft.com/en-us/cpp/build/reference/dumpbin-options?view=msvc-170) `/exports` on a dll file for another mod that got linked to me by hoppip in the modworkshop discord server.
The output looked like this:

```
    00000000 characteristics
    FFFFFFFF time date stamp
        0.00 version
           1 ordinal base
           8 number of functions
           8 number of names

    ordinal hint RVA      name

          1    0 000061C4 MODULE_LICENCE_DECLARATION
          2    1 000061C8 MODULE_SOURCE_CODE_LOCATION
          3    2 00006698 MODULE_SOURCE_CODE_REVISION
          4    3 000044E0 SBLT_API_REVISION
          5    4 00002760 SuperBLT_Plugin_Init_State
          6    5 00002770 SuperBLT_Plugin_PushLua
          7    6 00002780 SuperBLT_Plugin_Setup
          8    7 000027E0 SuperBLT_Plugin_Update
```

Now after adding a couple lines to my `lib.rs` file, I managed to make mine look the same (pretty much anyway).

```
    00000000 characteristics
    FFFFFFFF time date stamp
        0.00 version
           1 ordinal base
           8 number of functions
           8 number of names

    ordinal hint RVA      name

          1    0 0001836C MODULE_LICENCE_DECLARATION = _MODULE_LICENCE_DECLARATION
          2    1 000183A4 MODULE_SOURCE_CODE_LOCATION = _MODULE_SOURCE_CODE_LOCATION
          3    2 000183B0 MODULE_SOURCE_CODE_REVISION = _MODULE_SOURCE_CODE_REVISION
          4    3 000183B8 SBLT_API_REVISION = _SBLT_API_REVISION
          5    4 00001520 SuperBLT_Plugin_Init_State = _SuperBLT_Plugin_Init_State
          6    5 00001730 SuperBLT_Plugin_PushLua = _SuperBLT_Plugin_PushLua
          7    6 00001580 SuperBLT_Plugin_Setup = _SuperBLT_Plugin_Setup
          8    7 00001520 SuperBLT_Plugin_Update = _SuperBLT_Plugin_Init_State
```

Super BLT would accept this so far as well, however when trying to load the game would crash without any output in the console or log file. Clearly I must still be missing something.

## Down the \#define rabbit hole
After talking to Siri for a while I realized, that I should probably use the functions I defined in the `SuperBLT_Plugin_Setup` method parameters, as is done in the [original](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/src/msw32/init.cpp?ref_type=heads#L12-20). 
However the original does use something that I didn't define at all, which is the `all_lua_funcs_list`. 
This was defined in the [library's fptrs.h file](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h?ref_type=heads#L16) as a vector of a class `AutoFuncSetup`. 

Recreating this structure was trivial, however the way this vector was filled with data was anything but. (For the sake of my own sanity I will exclude the code here that is not C++ specific and only gets run in a C context.)

[`native-plugin-library/include/sblt_msw32_impl/fptrs.h:4-6`](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h?ref_type=heads#L4-6)
```c++ showLineNumbers{4}
#define IMPORT_FUNC(name, ret, ...) \
	extern "C" { extern ret (*name)(__VA_ARGS__); }
#else
```

Now this basically tells me, that everything run inside of an `IMPORT_FUNC` gets turned into an externally defined `C` function. So far so good.
However a bit later, we run into this:

[`native-plugin-library/include/sblt_msw32_impl/fptrs.h:27-30`](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h?ref_type=heads#L27-30)
```c++ showLineNumbers{27}
#undef IMPORT_FUNC
#define IMPORT_FUNC(name, ret, ...) \
	extern "C" { ret (*name)(__VA_ARGS__) = 0; } \
	AutoFuncSetup name ## _func_setup(#name, (void**) &name);
```

Now this is only ran in a context where `INIT_FUNC` is defined, however this is [defined in the initi.cpp of the same repository](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/src/msw32/init.cpp#L2) so I don't quite get the point of having this here separately but this whole file is a clusterfuck anyway.
This `IMPORT_FUNC` definition does the same as the first one, however also creates a new instance of `AutoFuncSetup` with the `name` and the memory address of the `name` cast to `void**`.

> As far as I'm aware, the reference is not used anywhere, so we can probably safely ignore that part.

This `IMPORT_FUNC` is also redefined by something called `CREATE_NORMAL_CALLABLE_SIGNATURE` in the same file. That looks like this:

[`native-plugin-library/include/sblt_msw32_impl/fptrs.h:1`](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h?ref_type=heads#L1)
```c++ showLineNumbers{1}
#define CREATE_NORMAL_CALLABLE_SIGNATURE(name, ret, a, b, c, ...) IMPORT_FUNC(name, ret, __VA_ARGS__)
```

Now then let's get to where this is used. Here's an example:

[`native-plugin-library/include/sblt_msw32_impl/fptrs.h:36`](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h?ref_type=heads#L36)
```c++ showLineNumbers{36}
IMPORT_FUNC(is_active_state, bool, lua_State *L)
```

This is quite clear, after running through this macro, it will turn into something like:

```c++
extern "C" {
	bool is_active_state(lua_State *L) = 0;
}
new AutoFuncSetup(*name, (void**) &name);
```

`AutoFuncSetup`'s constructor also pushes this value into the aforementioned vector.

[`native-plugin-library/include/sblt_msw32_impl/fptrs.h:20-22`](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h?ref_type=heads#L20-22)
```c++ showLineNumbers{20}
AutoFuncSetup(const char *name, void **ptr) : name(name), ptr(ptr) {
	all_lua_funcs_list.push_back(this);
}
```

However when reading this information from the vector in the `SuperBLT_Plugin_Setup` function, only the name is actually passed into the injected function, not the pointer `ptr`. 

[`native-plugin-library/src/msw32/init.cpp:14-17`](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/src/msw32/init.cpp?ref_type=heads#L14-17)
```c++ showLineNumbers{14}
for (const auto &func_ptr : all_lua_funcs_list) {
	AutoFuncSetup &func = *func_ptr;
	*func.ptr = get_exposed_function(func.name);
}
```

The function `get_exposed_function` is [injected internally by SuperBLT](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/plugins/plugins-w32.cpp#L87) and, as far as I can see, reads out the name that's passed to it, checks if it matches any SuperBLT defined functions and if not passes it further down into `get_lua_func`.

[`payday2-superblt/platforms/w32/plugins/plugins-w32.cpp:42-64`](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/plugins/plugins-w32.cpp#L42-64)
```c++ showLineNumbers{42}
static void* get_func(const char* name)
{
	string str = name;
	if (str == "pd2_log")
	{
		return &pd2_log;
	}
	else if (str == "is_active_state")
	{
		return &is_active_state;
	}
	else if (str == "luaL_checkstack")
	{
		return &luaL_checkstack;
	}
	else if (str == "lua_rawequal")
	{
		return &lua_rawequal;
	}
	
	return blt::platform::win32::get_lua_func(name);
}
```

`get_lua_func` then filters out getting the lua instance and functions to create a new instance. 
Then passes it even further down into `GetFunctionByName`.

[payday2-superblt/platforms/w32/platform.cpp:190-199](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/platform.cpp#L190-199)
```c++ showLineNumbers{190}
void* blt::platform::win32::get_lua_func(const char* name)
{
	// Only allow getting the Lua functions
	if (strncmp(name, "lua", 3)) return NULL;
	
	// Don't allow getting the setup functions
	if (!strncmp(name, "luaL_newstate", 13)) return NULL;
	
	return SignatureSearch::GetFunctionByName(name);
}
```

`GetFunctionByName` does a couple things but I'll need some setup to explain this.

[payday2-superblt/platforms/w32/signatures/signatures.cpp:411-425](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/signatures/signatures.cpp#L411-425)
```c++ showLineNumbers{411}
void* SignatureSearch::GetFunctionByName(const char* name)
{
	if (!allSignatures)
		return NULL;
		
	for (const auto& sig : *allSignatures)
	{
		if (!strcmp(sig.funcname, name))
		{
			return *(void**)sig.address;
		}
	}
	
	return NULL;
}
```

First of all it checks if a variable called `allSignatures` is truthy.
`allSignatures` is defined as a vector of `SignatureF` pointers and initialized as `NULL`.

[payday2-superblt/platforms/w32/signatures/signatures.cpp:308](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/signatures/signatures.cpp#L308)
```c++ showLineNumbers{308}
std::vector<SignatureF>* allSignatures = NULL;
```

`SignatureF` is defined as follows:

[payday2-superblt/platforms/w32/signatures/signatures.h:14-22](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/signatures/signatures.h#L14-22)
```c++ showLineNumbers{14}
struct SignatureF
{
	const char* funcname;
	const char* signature;
	const char* mask;
	int offset;
	void* address;
	SignatureVR vr;
};
```

The creation of these happens in the constructor of `SignatureSearch`, where the struct is constructed and then pushed into the `allSignatures` vector.

[payday2-superblt/platforms/w32/signatures/signatures.cpp:308-321](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/signatures/signatures.cpp#L308-321)
```c++ showLineNumbers{308}
std::vector<SignatureF>* allSignatures = NULL;

SignatureSearch::SignatureSearch(const char* funcname, void* adress, const char* signature, const char* mask, int offset, SignatureVR vr)
{
	// lazy-init, container gets 'emptied' when initialized on compile.
	if (!allSignatures)
	{
		allSignatures = new std::vector<SignatureF>();
	}
	
	SignatureF ins = {funcname, signature, mask, offset, adress, vr};
	allSignatures->push_back(ins);
}
```

And where is this called? Well this might look a little familiar from the Native Plugin Library

[payday2-superblt/platforms/w32/signatures/sigdef.h:6-9](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/signatures/sigdef.h#L6-9)
```c++ showLineNumbers{6}
#define CREATE_NORMAL_CALLABLE_SIGNATURE(name, retn, signature, mask, offset, ...) \
	typedef retn(*name ## ptr)(__VA_ARGS__); \
	name ## ptr name = NULL; \
	SignatureSearch name ## search(#name, &name, signature, mask, offset, SignatureVR_Both);
```

Wow. I'm not going to sugarcoat it, I've never felt more disdain for a language than I do right now. Who the fuck even made C++ this was such a dumb decision.

>[**Bjarne Stroustrup** ](https://en.wikipedia.org/wiki/Bjarne_Stroustrup)born 30 December 1950) is a Danish [computer scientist](https://en.wikipedia.org/wiki/Computer_scientist "Computer scientist"), most notable for the invention and development of the [C++](https://en.wikipedia.org/wiki/C%2B%2B "C++") programming language.

Well fuck you too Bjarne Stroustrup. You're making my life miserable.
Anyway moving on...

## Wrong assumptions
This basically means that the following signature from the Native Plugin Library:

[`native-plugin-library/include/sblt_msw32_impl/fptrs.h:40`](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h#L40)
```c++ showLineNumbers{40}
CREATE_NORMAL_CALLABLE_SIGNATURE(lua_call, void, "\x8B\x44\x24\x08\x8B\x54\x24\x04\xFF\x44\x24\x0C\x8D\x0C\xC5\x00", "xxxxxxxxxxxxxxxx", 0, lua_State*, int, int)
```

Would actually look this like this to SuperBLT:

```c++
new SignatureSearch(
	/*funcname*/lua_call, 
	/*address*/ 0x45646800,
	/*sig*/ "\x8B\x44\x24\x08\x8B\x54\x24\x04\xFF\x44\x24\x0C\x8D\x0C\xC5\x00",
	/*mask*/ "xxxxxxxxxxxxxxxx", 
	/*offset*/ 0,
	/*SignatureVR*/ 0
);
```

`SignatureVR` is just an enum defined in the same file as `SignatureF` and looks as follows:

[payday2-superblt/platforms/w32/signatures/signatures.h:7-12](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/signatures/signatures.h#L7-12)
```c++ showLineNumbers{7}
enum SignatureVR
{
	SignatureVR_Both,
	SignatureVR_Desktop,
	SignatureVR_VR
};
```

### Enlightenment
At least that's what I thought yesterday, however after looking at the code again and talking it over with Siri for a bit, I came to the conclusion that I'm fucking stupid. It's not even, that I was wrong about *how* the code works. I went wrong in thinking the `CREATE_NORMAL_CALLABLE_SIGNATURE` define worked the same in both `SuperBLT`s code and the Native Plugin Library.

Let's look at the first \#define from [[SuperBLT Native Plugin Template#Down the define rabbit hole|Down the define rabbit hole]] again:

[`native-plugin-library/include/sblt_msw32_impl/fptrs.h:4-6`](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h?ref_type=heads#L4-6)
```c++ showLineNumbers{4}
#define IMPORT_FUNC(name, ret, ...) \
	extern "C" { extern ret (*name)(__VA_ARGS__); }
#else
```

I correctly assumed, that `__VA_ARGS__` just took all the arguments from `...` and shoved them into the function call. However what I failed to realize is that, the `...` here is different from what I expected it to be, since `CREATE_NORMAL_CALLABLE_SIGNATURE` redefined `...` before passing it:

[`native-plugin-library/include/sblt_msw32_impl/fptrs.h:1`](https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h?ref_type=heads#L1)
```c++ showLineNumbers{1}
#define CREATE_NORMAL_CALLABLE_SIGNATURE(name, ret, a, b, c, ...) IMPORT_FUNC(name, ret, __VA_ARGS__)
```

What this means is, that in `CREATE_NORMAL_CALLABLE_SIGNATURE` all of the weird arguments that I struggled figuring out the meaning of in the last chapter, are actually completely ignored.
No Signature, no mask, and no offset.

So these are actually just normal external definitions, that I should be able to translate to rust as follows:

```c++ showLineNumbers{1}
CREATE_NORMAL_CALLABLE_SIGNATURE(lua_call, void, "\x8B\x44\x24\x08\x8B\x54\x24\x04\xFF\x44\x24\x0C\x8D\x0C\xC5\x00", "xxxxxxxxxxxxxxxx", 0, lua_State*, int, int)
```

[lua_call() definition](https://pgl.yoyo.org/luai/i/lua_call)
```rust
extern "C" {
	fn lua_call(L: *mut lua_State, nargs: c_int, nresults: c_int);
}
```

Well this really is kind of trivial. I should be able to just get all of these in and then get the dll to load right?

### Downfall
So remember how [[SuperBLT Native Plugin Template#Initial Victories|in the first chapter]], when I set the exports of the `dll` to match the ones from the Native Plugin Library code. That I then checked against the `Borderless Window Plugin`'s `dll`'s exports?

Yeah turns out that I missed one very crucial step but I'll get to that in a second.
First of all I'll talk about how I got to finding this out.

After all the effort of previous chapters, I now had hand defined every `lua` function that's defined in the Native Plugin Library/Template in rust but still couldn't get the game to not crash and burn when loading the `dll`. However sadly this also meant, that my `dll` didn't actually want to build. See I assume for C++ to build the library it doesn't actually link the external symbols at compile time?
I'm not sure though, I just know there's no `lua` `lib`/`dll` provided in there but the `dll` still builds.
The linker ran by Cargo however, actually required that I have a `lua` `dll` present at compile time, so I gave it one.

After comparing between the Borderless Window mod and mine however, I saw that the Borderless Window mod did in fact **not** have a `lua` `dll` included. So I know I did something wrong.
However I ignored this for now and simply commented out every use of `lua` functions I had in my code. I had to get the `dll` to load first, before I could worry about any of the linking issues.

I wasn't really sure at which point it was failing and a look through the source code didn't really give me an indication.

Here are my exported functions:

```rust
#[no_mangle]
pub extern "C" fn SuperBLT_Plugin_Setup(get_exposed_func: lua_access_func) {
    for &name in LUA_FUNCS.iter() {
        let cStringedName = CString::new(name).unwrap();
        get_exposed_func(cStringedName.as_ptr());
    }

    Plugin_Init();
}

#[no_mangle]
pub extern "C" fn SuperBLT_Plugin_Init_State(L: *mut lua_State) {
    Plugin_Setup_Lua(L);
}

#[no_mangle]
pub extern "C" fn SuperBLT_Plugin_Update() {
    Plugin_Update();
}

#[no_mangle]
pub extern "C" fn SuperBLT_Plugin_PushLua(L: *mut lua_State) {
    Plugin_PushLua(L);
}
```

Now people familiar with SuperBLT might already see something wrong here but for the the rest of you, I'm sure this looks as correct as can be. It did to me anyway.
After running down the functions in the SuperBLT repo for a while, I thought to myself that I'll need some logging.

So of course I did the most reasonable thing and download the source of SuperBLT.
Opened the folder in Visual Studio (since I kinda have to run this on Windows) and added some logging down the path the plugin load should take me.

[`payday2-superblt/src/plugins/plugins.cpp:125-149`](https://gitlab.com/znixian/payday2-superblt/-/blob/master/src/plugins/plugins.cpp#L125-149)
```c++ 
	PD2HOOK_LOG_LOG(string("Grabbing Plugin_init_State"));
	setup_state = (setup_state_func_t) ResolveSymbol("SuperBLT_Plugin_Init_State");
	if (!setup_state) throw string("Invalid dlhandle - missing setup_state func!");

	PD2HOOK_LOG_LOG(string("Grabbing Plugin_Update"));
	update_func = (update_func_t) ResolveSymbol("SuperBLT_Plugin_Update");
	PD2HOOK_LOG_LOG(string("Grabbing Plugin_PushLua"));
	push_lua = (push_lua_func_t) ResolveSymbol("SuperBLT_Plugin_PushLua");
}

void Plugin::AddToState(lua_State * L)
{
	PD2HOOK_LOG_LOG(string("Calling Plugin_Init_state"));
	setup_state(L);
}

void Plugin::Update(lua_State * L)
{
	PD2HOOK_LOG_LOG(string("Running Plugin_Update"));
	if (update_func)
		update_func(L);
}

int Plugin::PushLuaValue(lua_State * L)
{
	if(!push_lua)
		return 0;

	PD2HOOK_LOG_LOG(string("Running Plugin_PushLua"));
	return push_lua(L);
}
```

So on and so forth. Just to see where exactly it is, that I'm getting kicked out and crashing the game. Then I compiled the `WSOCK32.dll` and replaced the one in my Payday2 folder with my extra logging version.

And wouldn't you know it, I found the error pretty quickly... or well at least the place where I errored out.

[`payday2-superblt/src/InitiateState.cpp:712-743`](https://gitlab.com/znixian/payday2-superblt/-/blob/master/src/InitiateState.cpp#L712-743)
```c++
int luaF_load_native(lua_State* L)
{
	std::string file(lua_tostring(L, 1));

	try
	{
		blt::plugins::Plugin *plugin = NULL;
		blt::plugins::PluginLoadResult result = blt::plugins::LoadPlugin(file, &plugin);

		// TODO some kind of UUID system to prevent issues with multiple mods having the same DLL

		int count = plugin->PushLuaValue(L);

		if (count > 0)
			PD2HOOK_LOG_LOG("PushLuaValue result over 0 ");

		if (result == blt::plugins::plr_AlreadyLoaded)
		{
			lua_pushstring(L, "Already loaded");
		}
		else
		{
			lua_pushboolean(L, true);
		}

		PD2HOOK_LOG_LOG("running lua_insert");

		lua_insert(L, -1 - count); //fails on this

		PD2HOOK_LOG_LOG("post lua_insert");
		
		return count + 1;

	}
	catch (std::string err)
	{
		PD2HOOK_LOG_ERROR("GOT ERROR FROM luaF_load_native");
		luaL_error(L, err.c_str());
		return 0; // Fix the no-return compiler warning, but this will never be called
	}
}
```

My self inserted `PD2HOOK_LOG_LOG` statement `post lua_insert` never got reached.
My first thought was that this problem lies with `lua_insert()` but then it would fail on the Borderless Window mod as well. So next I checked if count was `> 0` but that log printed.

Then I went back into my own `dll`'s code.
`count` is set by executing my `dll`'s `PushLuaValue()` function, that I exposed.
So basically this snippet of code:

```rust
#[no_mangle]
pub extern "C" fn SuperBLT_Plugin_PushLua(L: *mut lua_State) {
    Plugin_PushLua(L);
}
```

Now let's hop to the definition of `Plugin_PushLua()`:

```rust
pub fn Plugin_PushLua(L: *mut lua_State) -> c_int {
    unsafe {
        lua_newtable(L);

        let helloWorldString = CString::new("Hellow, World!").unwrap();
        lua_pushstring(L, helloWorldString.as_ptr());
        let mystring = CString::new("mystring").unwrap();

        lua_setfield(L, -2, mystring.as_ptr());
    };

    return 1;
}
```

Well what could go wrong here? It couldn't have been the `unsafe` block, since I commented it out because of the linking issues I mentioned at the beginning of the chapter, however still no dice. 
I am returning 1 here, as is done in the Native Plugin Template so that can't really be the issue either.

[`native-plugin-template/src/main.cpp:20-65`](https://gitlab.com/SuperBLT/native-plugin-template/-/blob/master/src/main.cpp#L20-65)
```c++
int Plugin_PushLua(lua_State *L) {
	// Create a Lua table
	lua_newtable(L);
	// Add a hello-world string to it
	lua_pushstring(L, "Hello, World!");
	lua_setfield(L, -2, "mystring");
	// Add a C function to it
	lua_pushcfunction(L, say_hello);
	lua_setfield(L, -2, "myfunction");
	// Now return the table to Lua
	return 1;
}
```

I then commented out the `Plugin_PushLua()` function in my `SuperBLT_Plugin_PushLua()` and then it hit me.

```rust
#[no_mangle]
pub extern "C" fn SuperBLT_Plugin_PushLua(L: *mut lua_State) {
    Plugin_PushLua(L); //I never return the value here because i put a ;
}
```

I never actually returned the 1 from `Plugin_PushLua` so count didn't actually get set to anything in `SuperBLT`'s load function. I may be stupid. After changing the function signature a bit and actually returning a 1, I now had a `dll` that would actually load and not crash the game!

```rust
#[no_mangle]
pub extern "C" fn SuperBLT_Plugin_PushLua(L: *mut lua_State) -> c_int {
    Plugin_PushLua(L)
}
```

Now it was time to remove everything that I don't actually use from my code.
I tried using mlua again to run the lua functions, so I don't have to rely on the linkers and my jankily defined bindings. And guess what it worked.

I could now delete a solid 60 lines of `extern "C"` function definitions for all the lua functions

```rust
    pub fn lua_toboolean(L: *mut lua_State, index: c_int) -> c_int;
    pub fn lua_tointeger(L: *mut lua_State, index: c_int) -> lua_Integer;
    pub fn lua_tonumber(L: *mut lua_State, index: c_int) -> lua_Number;
    pub fn lua_tolstring(L: *mut lua_State, index: c_int, len: *mut c_size_t) -> *const c_char;
    pub fn lua_objlen(L: *mut lua_State, index: c_int) -> c_size_t;
    pub fn lua_touserdata(L: *mut lua_State, index: c_int) -> c_void;
```

Also for all the comparisons that I pulled in the Lua constants, I could get rid of these too:

```rust
#[repr(i8)]
enum LUA_TYPES {
    LUA_TNONE = -1,
    LUA_TNIL = 0,
    LUA_TBOOLEAN = 1,
    LUA_TLIGHTUSERDATA = 2,
    LUA_TNUMBER = 3,
    LUA_TSTRING = 4,
    LUA_TTABLE = 5,
    LUA_TFUNCTION = 6,
    LUA_TUSERDATA = 7,
    LUA_TTHREAD = 8,
}
```

I'm still not entirely sure if this block in `SuperBLT_Plugin_Setup()` is required or not but I'll try removing it and seeing how that works out.

```rust
for &name in LUA_FUNCS.iter() {
    let cStringedName = CString::new(name).unwrap();
    get_exposed_func(cStringedName.as_ptr());
}
```

Realistically I shouldn't need it, since all this does is look up the function name and returning a `void*` to the functions address from inside SuperBLT. But that `void*` isn't ever used after this, so I assume I could probably safely ignore this.

I'll stop here for now though and look more into this tomorrow.
I can absolutely safely resume this part of the code without breaking anything, so I'll do just that but now to something else equally as cancerous.

## Sisyphean Task
As it turns out I like to bash my head against the wall to slowly push through and that's exactly what I did here. Let me explain...

What I'm currently using to address lua is `*mut lua_State` a pointer to the `lua_State` that I get from `SuperBLT`s loader. This works and all but is kind of ugly to use because all the functions that use it are declared unsafe.
However `mlua` actually has it's own [`Lua` struct](https://docs.rs/mlua/latest/mlua/struct.Lua.html), which has much nicer, safe, functions associated with it. And this thing does have a function to [convert from a `*mut lua_State` ](https://docs.rs/mlua/latest/mlua/struct.Lua.html#method.init_from_ptr)...
I think you catch my drift.

Anyway I spent the next 2 days frantically trying to get it to convert the `lua_State` injected from `SuperBLT` into an `mlua` Lua struct. Sadly without any results.

I did however slowly go insane.
First of all running `init_from_ptr()` directly doesn't work at all. It just kind of crashes when trying to load the `dll`, of course again without an error message.

I then downloaded something called [`patch-crate`](https://crates.io/crates/patch-crate) (which i prefer to it's alternative 
[`cargo-patch`](https://crates.io/crates/cargo-patch), although I seem to have the minority opinion in that) to attempt to:
1. Debug my way through to see what the actual issue is in conversion
	and
2. To rectify any conversion issue to actually successfully create the Lua struct.

The result of this was, as said, a 2 day long slow descent into insanity where I would:
- add a `println!()`
- see where it stops printing
- check what the issue is
- dump half the lua stack
- point it to the right variable instead
- run it again
over and over and over again.

In-between every step I of course had to build the `dll` and put it into the Payday 2 mods folder and start Payday 2 to capture some console output.

Well as it turns out, after a lot of manual manipulation of the `mlua` and `mlua-sys` crates, that the conversion from the `lua_State` pointer provided by `SuperBLT` into an `mlua` Lua struct is not possible, due to the stack already having some kind of data on it before it's passed down to me.
The change file that `patch-crate` had given me at the end, showed `+ 1400` and `- 600` lines. 
All for naught.

Be that as it may, I've simply decided to write my own reasonably safe wrappers in time and only use the `mlua-sys` crate instead of the `mlua` crate for it's defined lua bindings. I couldn't use any of the actual `mlua` functionality anyway.

Here's an excerpt from one of the many console output snapshots i took when trying to run those heavily modified `mlua` and `mlua-sys` versions... for your amusement:

```
lua closure: pre rotate : 1
first if: 5
managed to lua rotate
checkstack: userdata
lua closure: pushlightuserdata
checkstack: userdata
lua closer: pcall
checkstack: function
lua closure: remove
checkstack: userdata
lua closure: ret 0
xxx
expect passed
running protect lua closure
lua closure: lua_gettop
checkstack: userdata
lua closure: lua_gettop
checkstack: userdata
lua closure: relax limit memory state
checkstack: function
checkstack: function
lua closure: pre rotate: 0
lua closure: pushlightuserdata
checkstack: userdata
lua closure: pcall
checkstack: function
lua closure: remove
checkstackL userdata
lua closure: ret 0
get ref_thread, false
get_gc_metatable : lua_rawgetp
checkstack: table
absindex: -10000
checkstack: userdata
checkstack: 0
this should be something: nil
this should not be true: true
wrapped_failure_mt_ptr
trying to push c func
false
```

If you really want to, you can probably vaguely see [where in the `init_from_ptr` function](https://github.com/mlua-rs/mlua/blob/8f3de8aa19ac7263297b080862c0a7a0ddb5de04/src/lua.rs#L449) I am by reading this but I wouldn't recommend trying to wrap your head around this if you're not proficient in rust and `lua`.

Anyway onto my next stupid venture, attempting to convert from a `void*` to a function signature in rust.

tl;cbawriting use `std::mem::transmute_copy()` with the pointer as an argument
Sadly that didn't work for the returned functions signatures that I said I could safely ignore at the end of [[#Downfall]] so whatever we ignore it until it becomes an issue `xd`

## Release?
This is currently on hold until the main project [[PD2 Heister's Haptics]] is finished. 
I'll probably have a bunch of useful, safe, function wrappers for the `lua` stuff by then and can consider a release of the plugin template, so I can save people from actually having to write `C++`, which is what I consider my mission in life.

# Update 2024/08/10
It's been a while hasn't it... I'm just here to write an update for this, since I picked the project back up.

## Broken
So downloading my repo and trying to compile and load it from a mod... didn't work.
I have no idea, genuinely, how it ever worked. With what I know now, it makes no sense to me, that I ever got it to work with the old setup.

Remember the thing I talked about right before the [[SuperBLT Native Plugin Template#Release?|Release?]] paragraph in here? Yeah I have no idea how I managed to fail casting the `void*` there to their respective function signatures. It worked just fine this time. In fact it worked so well, that I've done it for all of them.

The bigger issue for me was having to deep dive into macros to keep the project somewhat readable. I always kind of sucked at writing rust macros but I feel like I have an... at least decent handle on them now.

## Restored
Now I'm sure you're asking yourself how this all looks like... well let me show you.
The `SuperBLT_Plugin_Setup` function has evolved into this:

```rust
#[no_mangle]
pub extern "C" fn SuperBLT_Plugin_Setup(get_exposed_function: lua_access_func) {
    //We take out the logging function separately to spread our macros throughout the project
    let pd2_log_func_cstring = CString::new("pd2_log").unwrap();
    PD2HOOK_LOG.get_or_init(|| unsafe {
std::mem::transmute_copy(&get_exposed_function(pd2_log_func_cstring.as_ptr()))
    });
    
    //all panics will now produce error logs in mods/logs
    panic::set_hook(Box::new(|panic_info| {
        PD2HOOK_LOG_PANIC!("{}", panic_info);
    }));

    //this imports everything declared with IMPORT_FUNC or
    //CREATE_NORMAL_CALLABLE_SIGNATURE in SuperBLT's native plugin library
    //https://gitlab.com/SuperBLT/native-plugin-library/-/blob/master/include/sblt_msw32_impl/fptrs.h
    let mut superblt_instance = SUPERBLT.lock().unwrap();
    for func_name in SUPERBLT_EXPORTED_FUNCTIONS.into_iter() {
        let curr_func_name = CString::new(func_name.to_owned()).unwrap();
        superblt_instance.import_function(*func_name, get_exposed_function(curr_func_name.as_ptr()));
    }

    blt_funcs::plugin_init();
}
```

Now half of this stuff you haven't seen before but let me explain what's actually happening here really quickly.

### Logging
```rust
let pd2_log_func_cstring = CString::new("pd2_log").unwrap();
PD2HOOK_LOG.get_or_init(|| unsafe {
	std::mem::transmute_copy(&get_exposed_function(pd2_log_func_cstring.as_ptr()))
});
```

This part basically takes the string literal (`&str`) `"pd2_log"` and turns it into a `const char*` (`*const c_char` in rust). Then It initializes a [`OnceLock`](https://doc.rust-lang.org/nightly/std/sync/struct.OnceLock.html) called `PD2HOOK_LOG` by casting the `void*` (`*mut c_void` in rust), returned by SuperBLT's lookup function:

[`payday2-superblt/platforms/w32/plugins/plugins-w32.cpp:42-64`](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/plugins/plugins-w32.cpp?ref_type=heads#L42-64)
```c++
static void * get_func(const char* name)
{
	string str = name;
	if (str == "pd2_log")
	{
		return &pd2_log;
	}
	else if (str == "is_active_state")
	{
		return &is_active_state;
	}
	else if (str == "luaL_checkstack")
	{
		return &luaL_checkstack;
	}
	else if (str == "lua_rawequal")
	{
		return &lua_rawequal;
	}
	return blt::platform::win32::get_lua_func(name);
}
```

to the expected function signature, which is defined on `PD2HOOK_LOG` itself:

```rust
pub static PD2HOOK_LOG: OnceLock<
    fn(message: *const c_char, level: c_int, file: *const c_char, line: c_int),
> = OnceLock::new();
```

Which is just a direct translation of the `pd2_log` function found in the same file:

[`payday2-superblt/platforms/w32/plugins/plugins-w32.cpp:14-35`](https://gitlab.com/znixian/payday2-superblt/-/blob/master/platforms/w32/plugins/plugins-w32.cpp?ref_type=heads#L14-35)
```c++
static void pd2_log(const char* message, int level, const char* file, int line)
{
	using LT = pd2hook::Logging::LogType;
	char buffer[256];
	sprintf_s(buffer, sizeof(buffer), "ExtModDLL %s", file);
	switch ((LT)level)
	{
	case LT::LOGGING_FUNC:
	case LT::LOGGING_LOG:
	case LT::LOGGING_LUA:
		PD2HOOK_LOG_LEVEL(message, (LT)level, buffer, line, FOREGROUND_RED, FOREGROUND_BLUE, FOREGROUND_INTENSITY);
		break;
	case LT::LOGGING_WARN:
		PD2HOOK_LOG_LEVEL(message, (LT)level, buffer, line, FOREGROUND_RED, FOREGROUND_GREEN, FOREGROUND_INTENSITY);
		break;
	case LT::LOGGING_ERROR:
		PD2HOOK_LOG_LEVEL(message, (LT)level, buffer, line, FOREGROUND_RED, FOREGROUND_INTENSITY);
		break;
	}
}
```

After all that I can simply use pd2_log the same way they would. Although they don't really use this `pd2_log` function internally but I digress. I then created macros that are usable globally, which use this function in the same way I would use, for example `PD2HOOK_LOG_LOG`, in the `C++` version of the Native Plugin Template.

Let me show a single example for brevity:

The base of all the logging functions is a macro called `PD2HOOK_LOG_LEVEL`, which is only vaguely related to it's `C++` counterpart.

```rust
macro_rules! PD2HOOK_LOG_LEVEL {
    ($level:path; $($arg:tt)*) => {
        let log_message_cstring = CString::new(std::fmt::format(format_args!($($arg)*))).unwrap();
        let file_cstring = CString::new(file!()).unwrap();

        $crate::superblt::pd2_logger::PD2HOOK_LOG.get().unwrap()(
            log_message_cstring.as_ptr(),
            $level as std::ffi::c_int,
            file_cstring.as_ptr(),
            line!() as c_int,
        )
    };
```

I have to use macros for this, because otherwise I will not get the correct filename and line number from the `file!()` and `line!()` macros. Since these are expanded in place during compilation.

And here's `PD2HOOK_LOG_LOG` using that base macro:

```rust
macro_rules! PD2HOOK_LOG_LOG {
    ($($arg:tt)*) => {
        $crate::superblt::pd2_logger::PD2HOOK_LOG_LEVEL!(
            $crate::superblt::pd2_logger::LogType::LOGGING_LOG;
            $($arg)*
        )
    };
}
```

Without explaining too much how macros work, I'd like to talk about a trick I pulled here to make the Log function properly usable. `$($arg:tt)*` is *more or less* equivalent to C++ `VA_ARGS`. The definition of `log_message_cstring` basically just takes all arguments passed and treats them the same way the built-in `format!()` macro would. I'm sure you can see the resemblance here

[`std::format`](https://doc.rust-lang.org/1.80.1/src/alloc/macros.rs.html#123-128)
```rust
macro_rules! format {
    ($($arg:tt)*) => {{
        let res = $crate::fmt::format($crate::__export::format_args!($($arg)*));
        res
    }}
}
```

This then allows me to use `PD2HOOK_LOG_LOG` like this for example:

```rust
PD2HOOK_LOG_LOG!("Connecting to address: {}", ip_addr);
```

And have it produce a console output like this:

```
04:37:25 PM Log:  (ExtModDLL src\haptics\connector.rs:53) Connecting to address: ws://127.0.0.1:12345
```

### Panic
```rust
panic::set_hook(Box::new(|panic_info| {
    PD2HOOK_LOG_PANIC!("{}", panic_info);
}));
```

This is more or less self explanatory, if you're aware of what [`panic::set_hook`](https://doc.rust-lang.org/std/panic/fn.set_hook.html) does.
Basically, every time the application would panic (*more or less* equivalent to throwing an unhandled exception), instead of producing a standard error log, it will instead use one of my `PD2HOOK_LOG` logging functions to output to the SuperBLT developer console.

`panic_info` is just whatever the usual panic message would be.

### Function Signature Import
```rust
let mut superblt_instance = SUPERBLT.lock().unwrap();
for func_name in SUPERBLT_EXPORTED_FUNCTIONS.into_iter() {
	let curr_func_name = CString::new(func_name.to_owned()).unwrap();
        superblt_instance.import_function(*func_name, get_exposed_function(curr_func_name.as_ptr()));
}
```

Now for the real meat and potatoes.
Let's talk about `SUPERBLT` first. `SUPERBLT` is a struct that holds a `HashMap`, basically a key-value pair unsorted list, that associates a function name with a pointer to that function. Now this list starts out empty, obviously and gets filled right here in the `Init` function.

This is the basic definition of the `SuperBLT` struct.

```rust
#[derive(Clone, Default)]
pub struct SuperBLT {
	function_list: HashMap<String, *mut c_void>,
}
```

It also implements `Send` because I need to use it across threads later on but let's ignore all that stuff for now. The only other thing defined here is two functions, to add to this private `HashMap`.

```rust
impl SuperBLT {
    pub fn import_function(&mut self, function_name: &str, function_pointer: *mut c_void) {
        self.function_list
            .insert(function_name.into(), function_pointer);
    }

    pub fn get_function(&self, function_name: &str) -> &*mut c_void {
        self.function_list.get(function_name).unwrap_or(&null_mut())
    }
}
```

Now `SUPERBLT` is the same as `PD2HOOK_LOG` in the sense that it's just a globally unique instance of the `SuperBLT` struct. It's defined as follows:

```rust
pub static SUPERBLT: LazyLock<Mutex<SuperBLT>> = LazyLock::new(|| Mutex::new(SuperBLT::default()));
```

The reason I use a [`LazyLock`](https://doc.rust-lang.org/std/sync/struct.LazyLock.html) here instead of a [`OnceLock`](https://doc.rust-lang.org/std/sync/struct.OnceLock.html) is that I need to do the default initialization in place, before trying to insert stuff into it. It also makes accessing the `SUPERBLT` instance less messy in code later.

Now `SUPERBLT_EXPORTED_FUNCTIONS` is simply an array of string literals (`&str`) that contains all of the SuperBLT internal and lua functions that I could possible import. Here's a **very** abbreviated version:

```rust
pub const SUPERBLT_EXPORTED_FUNCTIONS: &[&str] = &[
    // superblt special handling
    "is_active_state",
    "luaL_checkstack",
    "lua_rawequal",

    // direct lua export
    "lua_call",
    "lua_pcall",
    "lua_gettop",
    "lua_settop",
    "lua_toboolean",
    // ...
];
```

Of course I exclude `pd2_log` here, because I import it separately, as mentioned in [[SuperBLT Native Plugin Template#Logging|Logging]].
Anyway, I iterate over every string here, turn it into a `const char*`, then shove it into the function that's injected by SuperBLT, and save the returned `void*` as well as the name of the function itself into the `SuperBLT` struct's `HashMap`.

#### Casting the function pointers
As mentioned before though, I need to cast these pointers to the actual function signature with [`std::mem::transmute_copy`](https://doc.rust-lang.org/std/mem/fn.transmute_copy.html). There's many ways to do this. Initially I wrote a ton of functions inside of the `impl` block of my `SuperBLT` struct, which looked something like this:

```rust
pub fn lua_call(&self, L: *mut lua_State, nargs: c_int, nresults: c_int) {
  let actual_func: fn(L: *mut lua_State, nargs: c_int, nresults: c_int) = unsafe {
    std::mem::transmute_copy(
      &self
        .function_list
        .get("lua_call")
        .unwrap(),
      )
    };

  actual_func(L, nargs, nresults)
}
```

But after writing like idk 10 of these I thought
- This is fucking awful I don't want to write a million of these
- I should cache the cast so I don't have to cast this every time I call the function

Those two points made me look into rust macros more, to write something similar to what the SuperBLT devs used with the `CREATE_NORMAL_CALLABLE_SIGNATURE` pre-processor macro.

After like 3 iterations of this, I present to you the final macro I used:

```rust
macro_rules! create_blt_callable {
    ($func_name:ident, $($param:tt: $ty:ty), *; $ret:ty) => {
        pub fn $func_name(&self, $($param:$ty), *) -> $ret {
            static ACTUAL_FUNC: std::sync::OnceLock<fn($($param:$ty), *) -> $ret> = std::sync::OnceLock::new();
            ACTUAL_FUNC.get_or_init(|| unsafe {std::mem::transmute_copy(
                self.get_function(stringify!($func_name))
            )})($($param), *)
        }
    };
}
```

It takes a `func_name` which is self explanatory, a variable number of comma separated parameters defined as `param_name: type` via `$($param:tt: $ty:ty), *`, and a return type, which is to be defined after a semicolon with the last `; $ret:ty` part.

Now with the help of this macro I was able to simplify my previous function definitions to just this:

```rust
create_blt_callable!(lua_call, L: *mut lua_State, nargs: c_int, nresults: c_int; ());
```

`()` in rust is equivalent to `void`. Although later on i let the macro have `$ret:ty` as an optional argument, so that part wasn't necessary anymore either. Now my `impl` for the `SuperBLT` struct looks as follows:

```rust
impl SuperBLT {
	create_blt_callable!(lua_call, L: *mut lua_State, nargs: c_int, nresults: c_int);
	create_blt_callable!(lua_pcall, L: *mut lua_State, nargs: c_int, nresults: c_int, errfunc: c_int; c_int);
	create_blt_callable!(lua_gettop, L: *mut lua_State; c_int);
	create_blt_callable!(lua_settop, L: *mut lua_State, index: c_int);
	create_blt_callable!(lua_toboolean, L: *mut lua_State, index: c_int; c_int);
	// ...
}
```

Of course there's a lot more of these but this sure simplified the entire process. Plus the macro would cache all of them in a [`OnceLock`](https://doc.rust-lang.org/std/sync/struct.OnceLock.html) too so I killed two birds with one stone.

There's a whole lot of functions that are not exported by SuperBLT though, a lot of them `luaL` functions. These I'd have to re-implement myself, but that's a trivial issue after getting all of this to work.

All that was left now... was to actually get Haptics to work as intended.
The Template **will** be released once we're done with `Heister's Haptics` and until then we'll probably be implementing a lot more stuff that isn't exported. So far it's the bare minimum.
Thank you for reading the update though, if you did. 

I noticed this site is one of the first couple results that come up when you google some specific keywords so, sorry you had to read all of this rambling, but also you're welcome if you learned something. I hope you were entertained anyway.