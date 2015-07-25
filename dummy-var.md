#虚变量

当一个方法返回多个值时，有些返回值有时候用不到，要是声明很多变量来一一接收，显然不太合适（不是不能）。 lua 提供了一个虚变量(dummy variable)，以单个下划线（“_”）来命名，用它来丢弃不需要的数值，仅仅起到占位的作用。

> 看一段示例代码：

```lua
-- string.find (s,p) 从 string 变量 s 的开头向后匹配 string 
-- p ，若匹配不成功，返回 nil ，若匹配成功，返回第一次匹配成功
-- 的起止下标。

local start, finish = string.find("hello", "he") -- start 值为起始下标， finish
                                                 -- 值为结束下标
print ( start, finish )                          -- 输出 1   2

local start = string.find("hello", "he")      -- start 值为起始下标
print ( start )                               --输出 1


local _,finish = string.find("hello", "he")   --采用虚变量（即下划线），接收起
                                              --始下标值，然后丢弃， finish 接收
                                              --结束下标值
print ( finish )                              --输出 2
```

代码倒数第二行，定义了一个用 local 修饰的`虚变量`（即 单个下划线）。使用这个虚变量接收 string.find()第一个返回值，静默丢掉，这样就直接得到第二个返回值了。

虚变量不仅仅可以被用在返回值，还可以用在迭代、函数输入等。

> 在 for 循环中的使用：

```lua
local t = {1, 3, 5}

for i,v in ipairs(table_name) do
    print(i,v)
end

for _,v in ipairs(table_name) do
    print(v)
end

```

> 在函数定义中的使用：

```lua
local _M = { _VERSION = '0.04' }
local mt = { __index = _M }

function _M.new(_, param)
  local self = {
        param=param
      }

  return setmetatable(self, mt)
end

-- ...

return _M

```


