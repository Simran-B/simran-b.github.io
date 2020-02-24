---
layout: post
title: Escaping a literal question mark in PowerShell
description: It isn't as straightforward as you may think for cmdlets which support wildcards
#categories: [PowerShell, Scripting] -- ends up in URL!
tags:
  - PowerShell
  - wildcards
  - escaping
---
I came across some PowerShell code like `Get-Process | ?{ $_.ProcessName -Match "^ex.*" }`
but didn't know what `?{ … }` does. I only knew `%{ … }`, which is the
shorthand notation of a `For-Each` where `$_` refers to the current element.
So my journey began.

## The Get-Alias cmdlet

A quick internet search for _powershell shorthand notations_ lead me to the
blog post [Weekend Scripter: Deciphering Windows PowerShell Shorthand][1].
The first thing it mentions is `Get-Alias` which lets you look up shorthands:

```powershell
Get-Alias iex       # positional argument
Get-Alias -Name iex # named argument
```

```
CommandType  Name                      Version  Source
-----------  ----                      -------  ------
Alias        iex -> Invoke-Expression
```

It also works the other way around, looking up the shorter aliases of commands:

```powershell
Get-Alias -Definition Invoke-Expression
```

```
CommandType  Name                      Version  Source
-----------  ----                      -------  ------
Alias        iex -> Invoke-Expression
```

## Too many alias lookup results

When I tried `Get-Alias ?` it answered my question (`?` is equivalent to
`Where-Object`), but to my surprised some other single character aliases were
listed too:

```
CommandType  Name                 Version  Source
-----------  ----                 -------  ------
Alias        % -> ForEach-Object
Alias        ? -> Where-Object
Alias        h -> Get-History
Alias        r -> Invoke-History
```

So there is obviously some special meaning to the question mark. However, the
PowerShell documentation [About Special Characters][2] does not list it in the
table of recognized escape sequences. I gave it a shot and tried
```Get-Alias `?```
anyway, but the result remained the same. I also tried a few other common
escaping patterns such as `Get-Alias \?`, wrapped in quotes, but it either
returned the same list or nothing.

## Escaping wildcard characters

The [Get-Alias documentation][3] mentions that you can

> Enter […] a pattern, such as "s*". Wildcards are permitted.

It doesn't say anything about `?` though. The page I found [About Wildcards][4]
lists the question mark together with the other supported wildcards `*` and
`[…]`, confirms that it _matches one character in that position_, but does not
contain any information about how to escape them.

My search continued to **Supporting Wildcard Characters in Cmdlet Parameters**,
which has a section about [Handling Literal Characters in Wildcard Patterns][5]:

> If the wildcard pattern you specify contains literal characters that should
> not be interpretted as wildcard characters, use the backtick character (`) as
> an escape character. When you specify literal characters in the PowerShell
> API, use a single backtick. When you specify literal characters at the
> PowerShell command prompt, **use two backticks**.

So ```Get-Alias ``?``` it is:

```
CommandType  Name               Version  Source
-----------  ----               -------  ------
Alias        ? -> Where-Object
```

## What is the PowerShell API?!

I couldn't find any meaningful information about the mentioned _PowerShell API_,
in which case you need a single backtick to escape `?` only. I arrived at the
conclusion that it refers to the usage of PowerShell in a .NET language such as
C#. The following code for a small console demo application supports my theory:

```csharp
using System;
using System.Management.Automation;

namespace TestPowershellApi
{
    class Program
    {
        static void Main(string[] args)
        {
            PowerShell ps = PowerShell
                .Create()
                .AddCommand("Get-Alias")
                .AddArgument("`?");            // positional argument
                //.AddParameter("Name", "`?"); // named argument

            foreach (PSObject result in ps.Invoke())
            {
                Console.WriteLine("{0} -> {1}",
                    result.Members["Name"].Value,
                    result.Members["Definition"].Value
                );
            }
        }
    }
}
```

```
? -> Where-Object
```

Note that there are also [PowerShell reference assemblies][6] available as
separate packages for certain use cases, but the built-in
`System.Management.Automation.PowerShell` is all you need if you want to
[execute PowerShell commands][7] in C#.

## Quadruple escaping for Invoke-Expression

In the (rare?) case that you want to evaluate a command or expression string
in a PowerShell script or the PowerShell command prompt, then you need not 1,
not 2, but 4 backticks:

```powershell
Invoke-Expression "Get-Alias ````?"
```

My assumption is that one level of escaping is necessary for the command line
itself, and the other because everything that is entered by the user is a
string at first. For example, `40+2` is a sequence of characters. However, it
gets evaluated to `42`. This requires parsing and casting the digits to numbers.
If every input is implicitly converted, then it could be necessary to escape a
single backtick with another one in order not to consume one and interpret the
subsequent character as wildcard instead of as literal.

## Summary

`?` is an alias of `Where-Object`, which [selects objects from a collection][8]
based on their property values. You can find that out with `Get-Alias`. To look
up just this alias and not every single-character alias, then you must escape
the question mark:

- **Single backtick** if you use the [PowerShell API](#what-is-the-powershell-api),
  such as using `System.Management.Automation` in C#:

  ```csharp
  PowerShell.Create().AddCommand("Get-Alias").AddArgument("`?")
  ```

- **Two backticks** in PowerShell scripts and the command prompt in general:

  ```powershell
  Get-Alias ``?
  ```

- **Four backticks** if it's string that you want to run as code:

  ```powershell
  Invoke-Expression "Get-Alias ````?"
  ```

The following commands are identical:

```powershell
Get-Process | ?{ $_.ProcessName -Match "^ex.*" }
Get-Process | Where-Object { $_.ProcessName -Match "^ex.*" }
Get-Process | Where-Object ProcessName -Match "^ex.*"
```

1. Script block format with shorthand
2. Script block format
3. Comparison statement format
4. Bonus: very short variant with slightly [different behavior][9]:

```powershell
ps | Where ProcessName -M "^ex.*"
```

[1]: https://devblogs.microsoft.com/scripting/weekend-scripter-deciphering-windows-powershell-shorthand/
[2]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_special_characters?view=powershell-7
[3]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-alias?view=powershell-7
[4]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_wildcards?view=powershell-7
[5]: https://docs.microsoft.com/en-us/powershell/scripting/developer/cmdlet/supporting-wildcard-characters-in-cmdlet-parameters?view=powershell-7#handling-literal-characters-in-wildcard-patterns
[6]: https://stackoverflow.com/questions/56749083/can-i-use-powershell-class-in-net-standard-library-to-use-in-net-framework-pro
[7]: https://docs.microsoft.com/en-us/powershell/scripting/developer/hosting/adding-and-invoking-commands?view=powershell-7
[8]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/where-object?view=powershell-7
[9]: https://stackoverflow.com/questions/50956909/powershell-where-object-vs-where-method
