# 13 – Metatables and Metamethods

Usually, tables in Lua have a quite predictable set of operations. We can add key-value pairs, we can check the value associated with a key, we can traverse all key-value pairs, and that is all. We cannot add tables, we cannot compare tables, and we cannot call a table.

Metatables allow us to change the behavior of a table. For instance, using metatables, we can define how Lua computes the expression `a+b`, where `a` and `b` are tables. Whenever Lua tries to add two tables, it checks whether either of them has a metatable and whether that metatable has an `__add` field. If Lua finds this field, it calls the corresponding value (the so-called *metamethod*, which should be a function) to compute the sum.

Each table in Lua may have its own *metatable*. (As we will see later, userdata also can have metatables.) Lua always create new tables without metatables:

```lua
t = {}
print(getmetatable(t))   --> nil
```

We can use `setmetatable` to set or change the metatable of any table:

```lua
t1 = {}
setmetatable(t, t1)
assert(getmetatable(t) == t1)
```

Any table can be the metatable of any other table; a group of related tables may share a common metatable (which describes their common behavior); a table can be its own metatable (so that it describes its own individual behavior). Any configuration is valid.

---

## 13.1 – Arithmetic Metamethods

In this section, we will introduce a simple example to explain how to use metatables. Suppose we are using tables to represent sets, with functions to compute the union of two sets, intersection, and the like. As we did with lists, we store these functions inside a table and we define a constructor to create new sets:

```lua
Set = {}

function Set.new (t)
    local set = {}
    for _, l in ipairs(t) do set[l] = true end
    return set
end

function Set.union (a,b)
    local res = Set.new{}
    for k in pairs(a) do res[k] = true end
    for k in pairs(b) do res[k] = true end
    return res
end

function Set.intersection (a,b)
    local res = Set.new{}
    for k in pairs(a) do
        res[k] = b[k]
    end
    return res
end
```

To help checking our examples, we also define a function to print sets:

```lua
function Set.tostring (set)
    local s = "{"
    local sep = ""
    for e in pairs(set) do
        s = s .. sep .. e
        sep = ", "
    end
    return s .. "}"
end

function Set.print (s)
    print(Set.tostring(s))
end
```

Now, we want to make the addition operator (`+`) compute the union of two sets. For that, we will arrange that all tables representing sets share a metatable and this metatable will define how they react to the addition operator. Our first step is to create a regular table that we will use as the metatable for sets. To avoid polluting our namespace, we will store it in the Set table:

```lua
Set.mt = {}    -- metatable for sets
```

The next step is to modify the `Set.new` function, which creates sets. The new version has only one extra line, which sets `mt` as the metatable for the tables that it creates:

```lua
function Set.new (t)   -- 2nd version
    local set = {}
    setmetatable(set, Set.mt)
    for _, l in ipairs(t) do set[l] = true end
    return set
end
```

After that, every set we create with `Set.new` will have that same table as its metatable:

```lua
s1 = Set.new{10, 20, 30, 50}
s2 = Set.new{30, 1}
print(getmetatable(s1))          --> table: 00672B60
print(getmetatable(s2))          --> table: 00672B60
```

Finally, we add to the metatable the so-called metamethod, a field `__add` that describes how to perform the union:

```lua
Set.mt.__add = Set.union
```

Whenever Lua tries to add two sets, it will call this function, with the two operands as arguments.
With the metamethod in place, we can use the addition operator to do set unions:

```lua
    s3 = s1 + s2
    Set.print(s3)  --> {1, 10, 20, 30, 50}
```

Similarly, we may use the multiplication operator to perform set intersection:

```lua
Set.mt.__mul = Set.intersection

Set.print((s1 + s2)*s1)     --> {10, 20, 30, 50}
```

For each arithmetic operator there is a corresponding field name in a metatable. Besides `__add` and `__mul`, there are `__sub` (for subtraction), `__div` (for division), `__unm` (for negation), and `__pow` (for exponentiation). We may define also the field `__concat`, to define a behavior for the concatenation operator.

When we add two sets, there is no question about what metatable to use. However, we may write an expression that mixes two values with different metatables, for instance like this:

```lua
s = Set.new{1,2,3}
s = s + 8
```

To choose a metamethod, Lua does the following: (1) If the first value has a metatable with an `__add` field, Lua uses this value as the metamethod, independently of the second value; (2) otherwise, if the second value has a metatable with an `__add` field, Lua uses this value as the metamethod; (3) otherwise, Lua raises an error. Therefore, the last example will call `Set.union`, as will the expressions `10 + s` and `"hy" + s`.

Lua does not care about those mixed types, but our implementation does. If we run the `s = s + 8` example, the error we get will be inside `Set.union`:

```terminal
bad argument #1 to `pairs' (table expected, got number)
```

If we want more lucid error messages, we must check the type of the operands explicitly before attempting to perform the operation:

```lua
function Set.union (a,b)
    if getmetatable(a) ~= Set.mt or
        getmetatable(b) ~= Set.mt then
        error("attempt to `add' a set with a non-set value", 2)
    end
    ...  -- same as before
```

---

## 13.2 – Relational Metamethods

Metatables also allow us to give meaning to the relational operators, through the metamethods `__eq` (*equality*), `__lt` (*less than*), and `__le` (*less or equal*). There are no separate metamethods for the other three relational operators, as Lua translates `a ~= b` to `not (a == b)`, `a > b` to `b < a`, and `a >= b` to `b <= a`.

(Big parentheses: Until Lua 4.0, all order operators were translated to a single one, by translating `a <= b` to `not (b < a)`. However, this translation is incorrect when we have a *partial order*, that is, when not all elements in our type are properly ordered. For instance, floating-point numbers are not totally ordered in most machines, because of the value *Not a Number* (*NaN*). According to the IEEE 754 standard, currently adopted by virtually every hardware, NaN represents undefined values, such as the result of `0/0`. The standard specifies that any comparison that involves NaN should result in false. That means that `NaN <= x` is always false, but `x < NaN` is also false. That implies that the translation from `a <= b` to not `(b < a)` is not valid in this case.)

In our example with sets, we have a similar problem. An obvious (and useful) meaning for `<=` in sets is set containment: `a <= b` means that a is a subset of b. With that meaning, again it is possible that both `a <= b` and `b < a` are false; therefore, we need separate implementations for `__le` (*less or equal*) and `__lt` (*less than*):

```lua
Set.mt.__le = function (a,b)    -- set containment
    for k in pairs(a) do
        if not b[k] then return false end
    end
    return true
end

Set.mt.__lt = function (a,b)
    return a <= b and not (b <= a)
end
```

Finally, we can define set equality through set containment:

```lua
Set.mt.__eq = function (a,b)
    return a <= b and b <= a
end
```

After those definitions, we are now ready to compare sets:

```lua
s1 = Set.new{2, 4}
s2 = Set.new{4, 10, 2}
print(s1 <= s2)       --> true
print(s1 < s2)        --> true
print(s1 >= s1)       --> true
print(s1 > s1)        --> false
print(s1 == s2 * s1)  --> true
```

Unlike arithmetic metamethods, relational metamethods do not support mixed types. Their behavior for mixed types mimics the common behavior of these operators in Lua. If you try to compare a string with a number for order, Lua raises an error. Similarly, if you try to compare two objects with different metamethods for order, Lua raises an error.

An equality comparison never raises an error, but if two objects have different metamethods, the equality operation results in false, without even calling any metamethod. Again, this behavior mimics the common behavior of Lua, which always classifies strings as different from numbers, regardless of their values. Lua calls the equality metamethod only when the two objects being compared share this metamethod.

---

## 13.3 – Library-Defined Metamethods

It is a common practice for some libraries to define their own fields in metatables. So far, all the metamethods we have seen are for the Lua core. It is the virtual machine that detects that the values involved in an operation have metatables and that these metatables define metamethods for that operation. However, because the metatable is a regular table, anyone can use it.

The `tostring` function provides a typical example. As we saw earlier, `tostring` represents tables in a rather simple format:

```lua
print({})      --> table: 0x8062ac0
```

(Note that `print` always calls `tostring` to format its output.) However, when formatting an object, `tostring` first checks whether the object has a metatable with a `__tostring` field. If this is the case, `tostring` calls the corresponding value (which must be a function) to do its job, passing the object as an argument. Whatever this metamethod returns is the result of `tostring`.

In our example with sets, we have already defined a function to present a set as a string. So, we need only to set the `__tostring` field in the set metatable:

```lua
Set.mt.__tostring = Set.tostring
```

After that, whenever we call print with a set as its argument, print calls `tostring` that calls `Set.tostring`:

```lua
s1 = Set.new{10, 4, 5}
print(s1)    --> {4, 5, 10}
```

The `setmetatable`/`getmetatable` functions use a metafield also, in this case to protect metatables. Suppose you want to protect your sets, so that users can neither see nor change their metatables. If you set a __metatable field in the metatable, `getmetatable` will return the value of this field, whereas `setmetatable` will raise an error:

```lua
Set.mt.__metatable = "not your business"

s1 = Set.new{}
print(getmetatable(s1))     --> not your business
setmetatable(s1, {})
    stdin:1: cannot change protected metatable
```
---

## 13.4 – Table-Access Metamethods

The metamethods for arithmetic and relational operators all define behavior for otherwise erroneous situations. They do not change the normal behavior of the language. But Lua also offers a way to change the behavior of tables for two normal situations, the query and modification of absent fields in a table.

---

### 13.4.1 – The `__index` Metamethod

I said earlier that, when we access an absent field in a table, the result is `nil`. This is true, but it is not the whole truth. Actually, such access triggers the interpreter to look for an `__index` metamethod: If there is no such method, as usually happens, then the access results in `nil`; otherwise, the metamethod will provide the result.

The archetypal example here is inheritance. Suppose we want to create several tables describing windows. Each table must describe several window parameters, such as position, size, color scheme, and the like. All these parameters have default values and so we want to build window objects giving only the non-default parameters. A first alternative is to provide a constructor that fills in the absent fields. A second alternative is to arrange for the new windows to *inherit* any absent field from a prototype window. First, we declare the prototype and a constructor function, which creates new windows sharing a metatable:

```lua
-- create a namespace
Window = {}
-- create the prototype with default values
Window.prototype = {x=0, y=0, width=100, height=100, }
-- create a metatable
Window.mt = {}
-- declare the constructor function
function Window.new (o)
    setmetatable(o, Window.mt)
    return o
end
```

Now, we define the `__index` metamethod:

```lua
Window.mt.__index = function (table, key)
    return Window.prototype[key]
end
```

After that code, we create a new window and query it for an absent field:

```lua
    w = Window.new{x=10, y=20}
    print(w.width)    --> 100
```

When Lua detects that w does not have the requested field, but has a metatable with an `__index` field, Lua calls this `__index` metamethod, with arguments w (the table) and `width` (the absent key). The metamethod then indexes the prototype with the given key and returns the result.

The use of the `__index` metamethod for inheritance is so common that Lua provides a shortcut. Despite the name, the `__index` metamethod does not need to be a function: It can be a table, instead. When it is a function, Lua calls it with the table and the absent key as its arguments. When it is a table, Lua redoes the access in that table. Therefore, in our previous example, we could declare `__index` simply as

```lua
Window.mt.__index = Window.prototype
```

Now, when Lua looks for the metatable's `__index` field, it finds the value of `Window.prototype`, which is a table. Consequently, Lua repeats the access in this table, that is, it executes the equivalent of

```lua
Window.prototype["width"]
```

which gives the desired result.

The use of a table as an `__index` metamethod provides a cheap and simple way of implementing single inheritance. A function, although more expensive, provides more flexibility: We can implement multiple inheritance, caching, and several other variations. We will discuss those forms of inheritance in [Chapter 16]().

When we want to access a table without invoking its `__index` metamethod, we use the rawget function. The call `rawget(t,i)` does a *raw* access to table `t`. Doing a raw access will not speed up your code (the overhead of a function call kills any gain you could have), but sometimes you need it, as we will see later.

---

### 13.4.2 – The `__newindex` Metamethod

The `__newindex` metamethod does for table updates what `__index` does for table accesses. When you assign a value to an absent index in a table, the interpreter looks for a `__newindex` metamethod: If there is one, the interpreter calls it *instead* of making the assignment. Like `__index`, if the metamethod is a table, the interpreter does the assignment in that table, instead of in the original one. Moreover, there is a raw function that allows you to bypass the metamethod: The call `rawset(t, k, v)` sets the value `v` in key `k` of table `t` without invoking any metamethod.

The combined use of `__index` and `__newindex` metamethods allows several powerful constructs in Lua, from read-only tables to tables with default values to inheritance for object-oriented programming. In the rest of this chapter we see some of these uses. Object-oriented programming has its own chapter.

---

### 13.4.3 – Tables with Default Values

The default value of any field in a regular table is `nil`. It is easy to change this default value with metatables:

```lua
function setDefault (t, d)
    local mt = {__index = function () return d end}
    setmetatable(t, mt)
end

tab = {x=10, y=20}
print(tab.x, tab.z)     --> 10   nil
setDefault(tab, 0)
print(tab.x, tab.z)     --> 10   0
```

Now, whenever we access an absent field in tab, its `__index` metamethod is called and returns zero, which is the value of d for that metamethod.

The `setDefault` function creates a new metatable for each table that needs a default value. This may be expensive if we have many tables that need default values. However, the metatable has the default value d wired into itself, so the function cannot use a single metatable for all tables. To allow the use of a single metatable for tables with different default values, we can store the default value of each table in the table itself, using an exclusive field. If we are not worried about name clashes, we can use a key like "`___`" for our exclusive field:

```lua
local mt = {__index = function (t) return t.___ end}
function setDefault (t, d)
    t.___ = d
    setmetatable(t, mt)
end
```

If we are worried about name clashes, it is easy to ensure the uniqueness of this special key. All we need is to create a new table and use it as the key:

```lua
local key = {}    -- unique key
local mt = {__index = function (t) return t[key] end}
function setDefault (t, d)
    t[key] = d
    setmetatable(t, mt)
end
```

An alternative approach to associating each table with its default value is to use a separate table, where the indices are the tables and the values are their default values. However, for the correct implementation of this approach we need a special breed of table, called *weak tables*, and so we will not use it here; we will return to the subject in [Chapter 17]().

Another alternative is to memoize metatables in order to reuse the same metatable for tables with the same default. However, that needs weak tables too, so that again we will have to wait until [Chapter 17]().

---

### 13.4.4 – Tracking Table Accesses

Both `__index` and `__newindex` are relevant only when the index does not exist in the table. The only way to catch all accesses to a table is to keep it empty. So, if we want to monitor all accesses to a table, we should create a *proxy* for the real table. This proxy is an empty table, with proper `__index` and `__newindex` metamethods, which track all accesses and redirect them to the original table. Suppose that t is the original table we want to track. We can write something like this:

```lua
t = {}   -- original table (created somewhere)

-- keep a private access to original table
local _t = t

-- create proxy
t = {}

-- create metatable
local mt = {
    __index = function (t,k)
    print("*access to element " .. tostring(k))
    return _t[k]   -- access the original table
    end,

    __newindex = function (t,k,v)
    print("*update of element " .. tostring(k) ..
                            " to " .. tostring(v))
    _t[k] = v   -- update original table
    end
}
setmetatable(t, mt)
```

This code tracks every access to t:

```lua
> t[2] = 'hello'
*update of element 2 to hello
> print(t[2])
*access to element 2
hello
```

(Notice that, unfortunately, this scheme does not allow us to traverse tables. The `pairs` function will operate on the proxy, not on the original table.)

If we want to monitor several tables, we do not need a different metatable for each one. Instead, we can somehow associate each proxy to its original table and share a common metatable for all proxies. A simple way to associate proxies to tables is to keep the original table in a proxy's field, as long as we can be sure that this field will not be used for other means. A simple way to ensure that is to create a private key that nobody else can access. Putting these ideas together results in the following code:

```lua
-- create private index
local index = {}

-- create metatable
local mt = {
    __index = function (t,k)
    print("*access to element " .. tostring(k))
    return t[index][k]   -- access the original table
    end,

    __newindex = function (t,k,v)
    print("*update of element " .. tostring(k) ..
                            " to " .. tostring(v))
    t[index][k] = v   -- update original table
    end
}

function track (t)
    local proxy = {}
    proxy[index] = t
    setmetatable(proxy, mt)
    return proxy
end
```

Now, whenever we want to monitor a table `t`, all we have to do is `t = track(t)`.

---

### 13.4.5 – Read-Only Tables

It is easy to adapt the concept of proxies to implement read-only tables. All we have to do is to raise an error whenever we track any attempt to update the table. For the `__index` metamethod, we can use a table---the original table itself---instead of a function, as we do not need to track queries; it is simpler and quite more efficient to redirect all queries to the original table. This use, however, demands a new metatable for each read-only proxy, with `__index` pointing to the original table:

```lua
function readOnly (t)
    local proxy = {}
    local mt = {       -- create metatable
    __index = t,
    __newindex = function (t,k,v)
        error("attempt to update a read-only table", 2)
    end
    }
    setmetatable(proxy, mt)
    return proxy
end
```

(Remember that the second argument to `error`, 2, directs the error message to where the update was attempted.) As an example of use, we can create a read-only table for weekdays:

```lua
days = readOnly{"Sunday", "Monday", "Tuesday", "Wednesday",
            "Thursday", "Friday", "Saturday"}

print(days[1])     --> Sunday
days[2] = "Noday"
stdin:1: attempt to update a read-only table
```
