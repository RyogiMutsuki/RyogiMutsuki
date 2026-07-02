---
title: C# 开发笔记：初识源生成器
date: 2026-06-30 12:50:42
updated: 2026-06-30 13:09:26
categories:
  - [后端笔记]
tags:
  - 后端笔记
  - 源生成器
  - 元编程
  - 预处理
draft: false
catalog: true
---
## 元编程是什么

!!什么是元编程？好吃吗？!!

元编程（Meta-Programming），是一种强大的技术。与传统编程不同的是，元编程是将代码视作数据，并允许代码进行处理、修改、生成等一系列操作。

## 为什么需要这些？

元编程最大的好处就是可以将一些重复性高的工作（例如 DTO 声明，属性映射，MVVM）交由机器执行，从而将人解放出来专注于实现机器不能处理的业务逻辑。

一个经典的例子就是 Spring Boot，其大量利用注解和反射对代码进行修改!!并创造了面向注解编程!!

## .NET Roslyn Source Generator

虽然 Spring Boot 面向注解编程非常高级好用，但是 Spring Boot 也有自己的缺点，因为其元编程是建立在反射上扫描注解和代码生成的。

而众所周知反射是有开销的，所以大量类扫描的开销可想而知。 !!这也就是为什么通常只有公司用这个，因为人家的服务器是一天 24 小时运转的。!!

既然反射有开销，那有什么办法解决呢？

当然有，既然反射有运行时开销，那么我们可以丢到编译期完成。 !!反正编译本身就比较吃 CPU 和时间，把 Source Generator 丢到编译期跑也不会死。!!

## 什么是 Source Generator？

要了解这个，我们需要先知道 C# 的编译器，也就是 Roslyn。

因为 Roslyn 设计之初采用了编译器即服务的设计理念，这也允许开发者调用 Roslyn 的 API 来对代码进行处理。

Source Generator 就是运行在 Roslyn 的一组分析程序，它将!!人类不可读的!!代码作为输入，以便你可以进行分析并根据预设规则进行处理，并产生输出。

!!并且你也确确实实可以用源生成器来分析代码，并根据预设规则创建警告和错误，这些内容也会被投射到 IDE 上。!!

## 快速上手

首先，我们需要新建一个控制台应用程序，然后在 Program.cs 写下以下内容。

```csharp
namespace ConsoleApp;

public partial class Program;

```

然后，我们需要再建一个类库项目，目标框架设置为 netstandard2.0。

使用以下命令添加必须的依赖。

```bash
dotnet add package Microsoft.CodeAnalysis.CSharp
dotnet add package Microsoft.CodeAnalysis.Analyzers # 这个是可选的，也就是装不装无所谓。
```

然后在 ProgramMainSourceGenerator.cs 写下以下内容。

```csharp
using System.Text;
using Microsoft.CodeAnalysis;

namespace SourceGenerators;

[Generator]
public class ProgramMainSourceGenerator: IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var sb = new StringBuilder();
        sb.AppendLine("namespace ConsoleApp;");
        sb.AppendLine("");
        sb.AppendLine("public partial class Program{");
        sb.AppendLine("    ");
        sb.AppendLine("    public static void Main(string[] args) => System.Console.WriteLine(\"Hello Tsukishiro Mei!\");");
        sb.AppendLine("}");
        context.RegisterPostInitializationOutput(sourceGeneratorContext =>
        {
            sourceGeneratorContext.AddSource("Program.g.cs", sb.ToString());
        });
    }
}

```

最后在需要生成代码的 csproj 写下以下内容

```xml
<ItemGroup>
    <ProjectReference Include="..\SourceGenerators\SourceGenerators.csproj" OutputItemType="Analyzer" 
                    ReferenceOutputAssembly="false" />
</ItemGroup>
```

最后，我们在终端 cd 到控制台项目，运行 dotnet run。

```bash
PS D:\TsukishiroMei\Projects\RoslynExample\ConsoleApp> dotnet run
D:\TsukishiroMei\Projects\RoslynExample\SourceGenerators\ProgramMainSourceGenerator.cs(7,14): warning RS1036: “SourceGenerators.ProgramMainSourceGenerator”: 包含分析器或源生成器的项目应指定属性“<EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>”
Hello Tsukishiro Mei!
```

是的很神奇，你的项目不仅没有编译失败，还多了一行输出。

## 发生了什么？

在此项目中，我们添加了一个源生成器，并让这个源生成器生成了一个静态的 Main 方法，接受 string[]。

现在，我们在引用 Source Generator 的项目 csproj PropertyGroup 添加 `<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>`，然后转到 obj/bin/构建配置/目标框架/generated/SourceGenerators/SourceGenerators.ProgramMainSourceGenerator 下查看生成的 Program.g.cs

你就会看到 Source Generator 为你生成了一个 partial class Program，里面包含了一个 Main 方法。

```csharp
namespace ConsoleApp;

public partial class Program{
    
    public static void Main(string[] args) => System.Console.WriteLine("Hello Tsukishiro Mei!");
}
```

回看我们之前写的 Source Generator，你会发现 Source Generator 输出的正是我们在 StringBuilder 写的东西。

这里我们用的是比较基础的 RegisterPostInitializationOutput，它假定生成的代码和你写的代码没有关系，所以也就不需要碰一些相对复杂的任务。

## 小结

~~本章我们了解了源生成器的基本概念，现在你可以去写源生成器了~~

:::TIP

如果你用的是 Rider，也可以在依赖项 -> 源生成器上看到 Source Generator 上看生成输出，不过我写这篇文章的时候用的是 VSCode。

:::