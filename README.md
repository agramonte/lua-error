## lua-error ##

Robust error handling for Lua which features:

* `try()`, `catch()`, `finally()` functions
* custom error objects


### Quick ###

```lua
-- import creates a base Error class and global funcs try(), catch(), finally()

local Error = require 'lua_error'


-- do this anywhere in your code:

try{
  function()
    -- make a call which could raise an error
  end,
  
  catch{
    function( err )
      -- handle the error
    end
  },
  
  finally{
    function()
      -- do some cleanup
    end
  }
}
```

> Note: the `catch{}` and `finally{}` are optional.



### Overview ###

The library is a culmination of several ideas found on the Internet put into a cohesive package. It was also inspired by the error handling in Python. (see References below)

There are two different components to this library which can either be used together or independently:

1. *Gobal functions*: `try`, `catch`, and `finally` which give structure
2. *Error object class*: which can be used by itself or subclassed for more refined errors


#### Lua Errors ####

The basic pieces of error handling built into Lua are the functions `error()` and `pcall()`. We only need to focus on `error()`, since that's what we use to raise an error condition in a program, like so:

```lua
error( "this is my error" )
```

that in turn will create something like this:

```
my_lua_file.lua:17: this is my error
stack traceback:
	[C]: in function 'error'
	/path_to_file/my_lua_file.lua:17: in main chunk
	[C]: in function 'require'
	?: in function 'require'
	/path_to_file/main.lua:104: in function 'main'
	/path_to_file/main.lua:110: in main chunk
```

In the error we can see our error string "`this is my error`" and the corresponding traceback.

As shown in our simple example, `error()` is often only used to create string-type errors. There are a couple of drawbacks to these types of errors in that they are:

1. they are fragile

  Is that string "`ProtocolError`" from my module or yours? If string "`out of data`" changes then my code will break

2. they are harder to represent other, finer-grained errors

  Like `error.overflow`, `app.error.protocol`, etc

Though one feature of `error()` which can help is that its argument can be anything, not just a string, so later we'll give it some Error objects.


#### try(), catch(), finally() ####

This function trio is the backbone of awesome error handling. The following is the basic structure using all three of the functions.

> Note: in the example below, `<func ref>` represents a function reference, for example: 
>
> `local func_ref = function() end`

```lua
try{
  <func ref>,
  
  catch{
    <func ref>
  },
  
  finally{
    <func ref>
  }
}
```

This format works because it takes advantage of Lua's dual-way to call functions, eg:

`hello()` or `hello{}`, the latter being equivalent to `hello( {} )`

So essentially this format is really a function `try()` which accepts a single `array` argument containing up to _three_ function references like so, `{ <func ref>, catch{}, finally{} }`.

Keep in mind that the terms `catch` and `finally` are themselves global functions just like `try`, and like `try` these each take a single `array` argument but contain only a single function like so `{ <func ref> }`.


Here are some alternate layouts showing the same thing:

```lua
flattened out:
try{ <func ref>, catch{ <func ref> }, finally{ <func ref> } }

same thing, including parens:
try({ <func ref>, catch({ <func ref> }), finally({ <func ref> }) })
```


#### Custom Errors ####

The objects in this framework use [`lua-objects`](https://github.com/dmccuskey/lua-objects) as the backbone.

Here's a quick example how to create a custom error type:

```lua
-- import module
local Error = require 'lua_error'

-- create custom error class
-- this class could be more complex,
-- but this is all we need for a custom error
local ProtocolError = newClass( Error, { name="Protocol Error" } )

-- raise an error
error( ProtocolError( "bad protocol" ) )
```

For more examples of custom errors, you can check out the unit tests or the projects [`dmc-wamp`](https://github.com/dmccuskey/dmc-wamp), [`lua-bytearray`](https://github.com/dmccuskey/lua-bytearray), etc.



#### Example ####

The following code snippet is a real-life example taken from [`dmc-wamp`](https://github.com/dmccuskey/dmc-wamp):

```lua
	try{
		function()
			self._session:onOpen( { transport=self } )
		end,

		catch{
			function(e)
				if type(e)=='string' then
					error( e )
				elseif e:isa( Error.ProtocolError ) then
					print( e.traceback )
					self:_bailout{
						code=WebSocket.CLOSE_STATUS_CODE_PROTOCOL_ERROR,
						reason="WAMP Protocol Error"
					}
				else
					print( e.traceback )
					self:_bailout{
						code=WebSocket.CLOSE_STATUS_CODE_INTERNAL_ERROR,
						reason="WAMP Internal Error ({})"
					}
				end
			end
		}
	}
```

In the `catch` you see that:
* first, we're checking to see if it's a regular string-type error. if so, re-raise the error since we only care about Error objects.
* second, by using the method `isa`, see if the error is type `ProtocolError`, bailout with protocol error.
* third, it's not an error we can handle, so bailout with an internal error.


###References###

* https://gist.github.com/cwarden/1207556
* http://www.lua.org/pil/8.4.html
* http://www.lua.org/wshop06/Belmonte.pdf
