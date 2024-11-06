P/Invoke（Platform Invocation Services）允许C#调用非托管代码，与Lua虚拟机直接交互，这是前提。

# Lua与C++、C交互


## 宿主环境调用lua

有一个虚拟的栈，通过
1. 压入函数`lua_getglobal(L, "my_function");`
2. 压入参数`lua_pushinteger`,`lua_pushnumber 等
3. 通过C++代码使用`lua_pcall`；
4. **返回值会被压入栈顶**。

![[Pasted image 20241031142718.png]]

## Lua调用 C++函数


``` c++
int my_cpp_function(lua_State* L) {
	int arg = lua_tointeger(L, 1); // 获取第一个参数 
	int result = arg * 2;
	lua_pushinteger(L, result);  // 进行计算  将结果压入栈中 
	return 1; // 返回值数量
}
```

# XLua与Unity

我们希望的是使用Lua的进行开发，那么应该会lua调用C#的逻辑会更多，首先第一步就是要将Untiy中的类型调用支持，首先需要有类型信息的元数据

要解决两个问题：
1. lua侧要能够获取到有哪些类信息，并能实现类对象和访问类方法（既包括Untiy引擎的，也包括我们自定义的类代码）
2. 当lua访问C#,C#要能够通过Lua虚拟机获取到实际的C#对象

前面第1点提到的就是注册C#对象到Lua虚拟机环境，类比下Lua是怎么模拟面向对象的，就是通过源表实现方法被所有类对象共享，然后属性私有；其实注册到Lua的C#类型信息也是通过源表存储C#类信息。


那么首先第一步配置哪些类需要注册到C#,可以通过打标签（Attribute)的方式来标记：
```c#
using System.Collections.Generic;
using XLua;

public static class XLuaConfig
{
    [LuaCallCSharp]
    public static List<System.Type> LuaCallCSharp = new List<System.Type>()
    {
        typeof(Player), // 将 Player 类添加到可被 Lua 调用的列表中
        typeof(Debug),  // 添加 Debug 以便在 Lua 中调用 Debug.Log
    };
}

```

XLua会为这些这些类生成一个C#代理类，在 `LuaEnv` 初始化过程中，会调用自动生成的 `__Register` 方法
``` c#
public static void __Register(RealStatePtr L)
{
    // 获取类型的元表信息
    ObjectTranslator translator = ObjectTranslatorPool.Instance.Find(L);
    Type type = typeof(MyClass);

    // 开始注册对象类型
    Utils.BeginObjectRegister(type, L, translator, 0, 2, 1, 1);

    // 注册方法
    Utils.RegisterFunc(L, Utils.METHOD_IDX, "MyMethod", _m_MyMethod);

    // 注册属性
    Utils.RegisterFunc(L, Utils.GETTER_IDX, "MyProperty", _g_get_MyProperty);
    Utils.RegisterFunc(L, Utils.SETTER_IDX, "MyProperty", _s_set_MyProperty);

    Utils.EndObjectRegister(type, L, translator, null, null, null, null, null);
}

```

- `BeginObjectRegister` 开始对象的注册过程，为 `MyClass` 类型创建一个元表。
- `RegisterFunc` 用于注册类型的方法和属性。这里注册了方法 `MyMethod` 和属性 `MyProperty` 的 `getter` 和 `setter`。
- `EndObjectRegister` 结束对象的注册，将元表写入 Lua 虚拟机，使得 Lua 可以访问这个 C# 类。

### Lua 调C#的过程，以实例化一个对象为例

首先前面提到的C#代理类（自动生成的，xxxWrap结尾），代理类会根据参数等情况调用实际的C#类的构造函数

在`AddObject`方法中，xLua会为新的C#对象生成一个唯一的ID，并将其存入对象映射表中。具体实现如下：``
``` c#
private int AddObject(object obj)
{
    int objectId = nextObjectId++; // 分配一个自增的唯一ID
    objMap[objectId] = obj;        // 将对象存入映射表中
    return objectId;
}


```

分配完对象ID后，xLua会将这个ID推入Lua栈中，并在Lua中将其表示为一个C#对象的代理表。代理表在Lua中表现为普通的table，但通过xLua的内部元表（metatable）机制，可以调用C#对象的方法和属性。
```c#
// Push方法中调用AddObject为对象分配ID并存储
public void Push(IntPtr L, object obj)
{
    int objectId = AddObject(obj);
    LuaAPI.xlua_pushcsobj(L, objectId, 0);//pushcsobt
}

```
C# 对象在 Lua 对应的就是一个 `userdata`,`ObjectTranslator`类的`GetObject`

`xlua_tocsobj_safe`和`xlua_tocsobj_fast`
