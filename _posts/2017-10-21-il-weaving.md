---
layout: post
title: IL weaving
subtitle: A look at IL weaving with Fody
---

# Intermediate Language
All .NET code gets compiled down into CIL (Common Intermediate Language) instead of machine code. This IL can be run in CLI (Common Language Infrastructure) compatible runtime environments such as the .NET or Mono runtimes. This means that the same code can be executed on a multitude of platforms without the need of compiling them against specific hardware.

When you change the generated intermediate language after the code has compiled, this is called IL weaving. Instead of manipulating the source code, you're manipulating the building blocks of .NET.

To get a feeling of what IL looks like, you can find a 'Hello World' class in C# below:

```csharp
using System;

namespace HelloWorld
{
  class MainClass
  {
    public static void Main(string[] args)
    {
      Console.WriteLine("Hello World!");
    }
  }
}
```

And the same class in IL:

~~~
.class private auto ansi beforefieldinit HelloWorld.MainClass
	extends [mscorlib]System.Object
{
	// Methods
	.method public hidebysig static
		void Main (
			string[] args
		) cil managed
	{
		// Method begins at RVA 0x2050
		// Code size 11 (0xb)
		.maxstack 8
		.entrypoint

		IL_0000: ldstr "Hello World!"
		IL_0005: call void [mscorlib]System.Console::WriteLine(string)
		IL_000a: ret
	} // end of method MainClass::Main

	.method public hidebysig specialname rtspecialname
		instance void .ctor () cil managed
	{
		// Method begins at RVA 0x205c
		// Code size 7 (0x7)
		.maxstack 8

		IL_0000: ldarg.0
		IL_0001: call instance void [mscorlib]System.Object::.ctor()
		IL_0006: ret
	} // end of method MainClass::.ctor

} // end of class HelloWorld.MainClass
~~~

### Why weaving?
A lot of codebases have similar statements everywhere but not all of them are easily refactored in to a method, base class or behaviour. By weaving this type of code you're able to remove a lot of repeating code, resulting in cleaner code and fast development.

It's used a lot for automatically adding method tracing, logging or to implement common .NET patterns like the INotifyPropertyChanged pattern. It can also be used to fix common code issues automatically: ensure items are disposed, using statements are used, ...

### Why not?
'Magic behaviour' is usually not seen as a positive in programming. For new people to a codebase it might be hard to understand what's going on if some code isn't actually written down but added at compile time.

As it can be hard to take care of all possible scenarios when writing IL code, it's possible an addin could introduce a bug in your codebase. This can be really hard to track down or, if not resolvable through the plugin, could mean you have to write the weaved code manually anyway. It's important to balance the ease of use and speed of coding with how comfortable you are with extensions changing the code you've written.

# Let's make something

Introducing [Fody](https://github.com/Fody/Fody), an framework which helps you weave .net assemblies. Fody takes care of the plumbing work so you can easily write new addins on top of Fody. A list of some great existing addins can be found on the github page

As example I'm creating a Fody addin here which will allow changing a boolean state for the duration of a method. This could for example be used to automatically set 'loading' or 'syncing' states.

### Approach

- Methods which want to have this behaviour should add a C# attribute. Our weaver kicks in if it detects a [AddState("PropertyName")] attribute on top of a method.
- Passed properties should be verified. If the property doesn't exist, we'll create it ourselves. If the property does exist, we'll validate if it can be used.
- Methods will be wrapped in a try/finally block. Setting our state to true at the first line, and resetting it to false in the finally block.


### Find all methods with our attribute

Finding all methods tagged with our attribute is fairly straightforward. Fody (through [Cecil](http://www.mono-project.com/docs/tools+libraries/libraries/Mono.Cecil/)) allows us to traverse in the code to get what we need. To get our methods we need to get all classes, get all methods inside of those classes and check on the customattributes of those methods.

When processing the methods, we keep track of the current type, the current method and the requested state property name.

```csharp
// get all available classes
var allClasses = ModuleDefinition
            .GetTypes()
            .Where(x => x.IsClass)
            .ToList();

// get all methods in the classes
var methods = allClasses
            .SelectMany(type => type.Methods)
            .Where(method =>
        {
            // only select the methods which have our AddState attribute defined
            var attribute = method.CustomAttributes
                  .FirstOrDefault(attr => attr.Constructor.DeclaringType.FullName == "State.Fody.AddStateAttribute");
            return attribute != null;
        });
```

### Validate state properties

At this point we have a list of methods linked to state property names. We need to verify the requested properties. We'll keep a reference to the property or field to make setting the value easier during weaving. During this verification step we check the following:

- A property with the name exists, is accessible and has the correct type: OK
- A property with the name exists but has no setter: NOK - currently not supported
- A property with the name exists but is not a boolean: NOK
- A field with the name exists, is accessible and has the correct type: OK
- A field with the name exists but is not a boolean: NOK
- There is no property or field with the name: OK

Items with NOK will throw an exception. This will fail the code compilation, showing a message what went wrong.

### Creating state properties

If properties are requested which don't exist yet we need to create them. Extra validation is required for the properties which need to be created. The same property creation could be requested multiple times. Inheritance also needs to be taken into account. If both a subclass and a baseclass want to make the same property we can disregard the subclass and just add it on the baseclass.

Once the properties are trimmed we can weave them in to the correct types. A property is in fact a reference to two methods, a setter method and a getter method. A backing field will be added as well. I copied to code for this below with some comments explaining what's going on.

```csharp
PropertyDefinition CreateProperty(TypeDefinition typeDefinition, string statePropertyName)
{
    PropertyDefinition propertyDefinition;
    var propertyType = ModuleDefinition.TypeSystem.Boolean;
    var voidType = ModuleDefinition.TypeSystem.Void;

    // create private backing field
    var fieldDefinition = new FieldDefinition($"<{statePropertyName}>k_BackingField", FieldAttributes.Private, propertyType);
    typeDefinition.Fields.Add(fieldDefinition);

    var parameterDefinition = new ParameterDefinition(propertyType);

    // create property
    var attributes = MethodAttributes.FamANDAssem | MethodAttributes.Family | MethodAttributes.HideBySig | MethodAttributes.SpecialName;

    // setup getter method
    var getMethod = new MethodDefinition("get_" + statePropertyName, attributes, propertyType)
    {
        IsGetter = true,
        SemanticsAttributes = MethodSemanticsAttributes.Getter
    };
    // setup setter method
    var setMethod = new MethodDefinition("set_" + statePropertyName, attributes, voidType)
    {
        IsSetter = true,
        SemanticsAttributes = MethodSemanticsAttributes.Setter
    };
    setMethod.Parameters.Add(parameterDefinition);

    // write the actual getter code
    var getter = getMethod.Body.GetILProcessor();
    // load 'this'
    getter.Emit(OpCodes.Ldarg_0);
    // get the value of our backing field
    getter.Emit(OpCodes.Ldfld, fieldDefinition);
    // return the value
    getter.Emit(OpCodes.Ret);

    // write the actual setter code
    var setter = setMethod.Body.GetILProcessor();
    // load 'this'
    setter.Emit(OpCodes.Ldarg_0);
    // load passed value
    setter.Emit(OpCodes.Ldarg, parameterDefinition);
    // store the value in our backing field
    setter.Emit(OpCodes.Stfld, fieldDefinition);
    // return
    setter.Emit(OpCodes.Ret);

    typeDefinition.Methods.Add(getMethod);
    typeDefinition.Methods.Add(setMethod);

    // hook up all property bits and add it to our type
    propertyDefinition = new PropertyDefinition(statePropertyName, PropertyAttributes.None, propertyType)
    {
        HasThis = true,
        GetMethod = getMethod,
        SetMethod = setMethod
    };

    typeDefinition.Properties.Add(propertyDefinition);
    return propertyDefinition;
}
```

### Weave in our state changes

We're finally at the point to weave in our state changes. The first thing we need to do is ensure our method has only one return statement. This is necessary for the try flow to work correctly. To keep this easy we create a new return statement which we add to the end change all others to a Br_S opcode to break to this instruction.

Next we insert our try/finally clause.

```csharp
// find our first method instruction
// => if the method is the constructor we need to skip the 'base' call instruction
var methodBodyFirstInstruction = methodDefinition.Body.Instructions.First();
if (methodDefinition.IsConstructor &&
    methodDefinition.Body.Instructions.Any(i => i.OpCode == OpCodes.Call))
{
    methodBodyFirstInstruction = methodDefinition.Body.Instructions.First(i => i.OpCode == OpCodes.Call).Next;
}
// full code to get return instruction omitted
var methodBodyReturnInstruction = methodBodyReturnInstructions.First();
// define the last line of code in the try block with the Leave_S opcode
var tryCatchLeaveInstructions = Instruction.Create(OpCodes.Leave_S, methodBodyReturnInstruction);
var finallyInstructions = new List<Instruction>()
{
    // actual code to execute in finally method would be added here
    // indicate end of finally clause
    Instruction.Create(Opcodes.Endfinally)
}

// tweak instruction order
processor.InsertBefore(methodBodyReturnInstruction, tryCatchLeaveInstructions);
processor.InsertBefore(methodBodyReturnInstruction, finallyInstructions);

// hook up finally handler
var handler = new ExceptionHandler(ExceptionHandlerType.Finally)
{
    TryStart = methodBodyFirstInstruction,
    TryEnd = tryCatchLeaveInstructions.Next,
    HandlerStart = finallyInstructions.First(),
    HandlerEnd = finallyInstructions.Last().Next
};

// add our handler to the method body
methodDefinition.Body.ExceptionHandlers.Add(handler);
```

At last we add our setter code. Setter instructions are created once with value 1 to set our state to true and once with value 0 to set our state to false. The true statements are inserted before the first method instruction. The false statements are inserted before the Endfinally Opcode.

```csharp
// simplified setter code
// => actual code changes depending on field or property
var setInstructions = new List<Instruction>
{
    Instruction.Create(OpCodes.Ldarg_0),
    // push an int32 value (0 for false, 1 for true)
    Instruction.Create(OpCodes.Ldc_I4, value),
    // call our setter
    Instruction.Create(OpCodes.Call, methodNode.PropertyReference)
};
```

### Dealing with async

Async/await is syntactic sugar that gets compiled down to Task code. A new nested class is made by the compiler which executes the async code and takes care of proper continuation when the result is available. This generated code looks like this. Note the AsyncStateMachine attribute linking to the created nested type.

```csharp
[AsyncStateMachine (typeof(Async.<TestAsync1>d__10))]
public Task TestAsync1 ()
{
  Async.<TestAsync1>d__10 <TestAsync1>d__ = new Async.<TestAsync1>d__10 ();
  <TestAsync1>d__.<>4__this = this;
  <TestAsync1>d__.<>t__builder = AsyncTaskMethodBuilder.Create ();
  <TestAsync1>d__.<>1__state = -1;
  AsyncTaskMethodBuilder <>t__builder = <TestAsync1>d__.<>t__builder;
  <>t__builder.Start<Async.<TestAsync1>d__10> (ref <TestAsync1>d__);
  return <TestAsync1>d__.<>t__builder.Task;
}
```

Our current try/finally weave will wrap this code without waiting for the task result. To fix this we'll have to look further into what gets generated for async methods.

Every async method creates a nested class of type [IAsyncStateMachine](https://msdn.microsoft.com/en-us/library/system.runtime.compilerservices.iasyncstatemachine(v=vs.110).aspx). This hase a MoveNext method which executes the actual code. This method always follows the same pattern.

As there's already a try/catch clausule to handle task exceptions we'll simply add the try/finally clause inside of this method instead of over the wrapping method.

```csharp
void IAsyncStateMachine.MoveNext ()
{
  try {
    int num = this.<>1__state;
    try {
      TaskAwaiter awaiter;
      if (num != 0) {
      awaiter = //some async code
      if (!awaiter.IsCompleted) {
        this.<>1__state = 0;
        this.<>u__1 = awaiter;
        Async.<TestAsync1>d__10 <TestAsync1>d__ = this;
        this.<>t__builder.AwaitUnsafeOnCompleted<TaskAwaiter, Async.<TestAsync1>d__10> (ref awaiter, ref <TestAsync1>d__);
        return;
      }
    } else {
      awaiter = this.<>u__1;
      this.<>u__1 = default(TaskAwaiter);
      this.<>1__state = -1;
    }
    awaiter.GetResult ();
    // Some sync code
    } catch (Exception exception) {
      this.<>1__state = -2;
      this.<>t__builder.SetException (exception);
      return;
    }
  this.<>1__state = -2;
  this.<>t__builder.SetResult ();
  }
}
```

Instead of operating on the passed method, as explained above, we'll fetch the nested type and find the 'MoveNext' method. By using this method as a target we can add a try finally clause to async code.

Most of the try/finally weaving code is the same as for the normal sync methods with some minor differences.

- Return statements don't need to be simplified as the method is already in a try/catch clausule. We know that there's only one return statement.
- We need to access our state property through a 'this' property since we're in a nested method
- Before setting state we need to check on our task state to avoid changing state when the task hasn't completed yet


# State.Fody

Complete code for this state plugin can be found on [github](https://michielsioen.be/2017-10-21-il-weaving/).

I'm planning to have a look at the following going forwards:

- create nuget package
- support for correctly keeping the state if changed in multiple submethods
- ensure thread-safe changing of the state
- proper unit tests
<br />
<br />
