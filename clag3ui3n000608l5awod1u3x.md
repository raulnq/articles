# Husky.Net: Git hooks made easy

During our day-to-day, usually, there are repetitive tasks we always do: formatting code, running unit tests, following commit policies, etc. Easy tasks, but we make them over and over again, which is a waste of time and effort. Luckily there is a way to automate those tasks, [Git hooks](https://git-scm.com/docs/githooks). In this post, we will explore what they are and then review [Husky.Net](https://alirezanet.github.io/Husky.Net/), a simple way to start using Git hooks under the .NET ecosystem.

## Git hooks

Git hooks provide a way to fire off custom scripts on different events (after and before), such as during commit, push or rebase, etc. There are two types of hooks:

- Client-side: Triggered by events in the local repository, such as when a developer commits code.
- Server-side: Run under the remote repository and are triggered by events such as pushes received.

## Useful client-side hooks

### pre-commit

Invoked by `git commit` before Git asks the developer for a commit message or generates a commit object. It takes zero arguments and exiting with a non-zero status aborts the commit operation. Used for static analysis, linting, spell-checks, and code style checks or run tests.

### prepare-commit-msg

Invoked by `git commit` after preparing the default commit message, just before launching the commit message editor. Arguments: 
- Name of the file that contains the commit log message. 
- The type of commit: `message`, `template`, `merge`, `squash`, or `commit`.
- Commit SHA1.

It is useful for editing the default message before the commit author sees it (e.g., adding the branch name to the commit message).

### commit-msg
Invoked by `git commit` after the user enters a commit message. This hook takes one parameter and is the path to a temporary file that contains the commit message. It is an appropriate place to validate the commit state or message to ensure compliance with a standard or to reject based on any criteria.

### post-commit
Invoked by `git commit`. It takes no parameter and runs after the commit operation is completed. This hook can be used to provide notifications.

### pre-push
Invoked by `git push` and can be used to prevent a push from taking place. The hook has two parameters, the remote name, and location(URL).

## Husky. NET

> Husky.Net offers a very simple way to start using git hooks or running certain tasks, write and run custom scripts and more ...

Let's start installing the tool:

```powershell
dotnet tool install --global Husky
```

Now, we will create a library to sum two numbers:

```powershell
dotnet new classlib -n calculator
dotnet new mstest -n test
dotnet new sln -n husky-sandbox
dotnet add tests/tests.csproj reference calculator/calculator.csproj
dotnet sln add --in-root calculator
dotnet sln add --in-root tests
``` 

Open the solution and go to the `calculator` project. Add a `Calculator.cs` file with the following content:

```csharp
public class Calculator
{
    public int Sum(int a, int b)
    {
        return a + b;
    }
}
``` 

Go to the `test` project and add a `CalculatorTests.cs` file as follows:

```csharp
[TestClass]
public class CalculatorTests
{
    [TestMethod]
    [DataRow(1, 2, 3)]
    [DataRow(2, 2, 4)]
    [DataRow(0, 0, 0)]
    [DataRow(-1, 1, 0)]
    public void sum_X_and_Y_should_be_Z(int x, int y, int z)
    {
        var sut = new Calculator();

        var r = sut.Sum(x, y);

        Assert.AreEqual(z, r);
    }
}
``` 

Run `dotnet husky add pre-commit -c "echo 'running pre-commit'"` to create our first git hook. The command is going to create a `.husky/pre-commit` file with the content below:

```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

## husky task runner examples -------------------
## Note : for local installation use 'dotnet' prefix. e.g. 'dotnet husky'

## run all tasks
#husky run

### run all tasks with group: 'group-name'
#husky run --group group-name

## run task with name: 'task-name'
#husky run --name task-name

## pass hook arguments to task
#husky run --args "$1" "$2"

## or put your custom commands -------------------
#echo 'Husky.Net is awesome!'

echo 'running pre-commit'
``` 

Let's do some code formatting with [dotnet-format](https://github.com/dotnet/format). Add `dotnet format -v detailed` at the end of the file and run `git commit -m "testing git hooks" --allow-empty`. We will get an output like this:

```
running pre-commit
  The dotnet runtime version is '6.0.8'.
  Formatting code files in workspace 'C:\Source\husky-sandbox\husky-sandbox.sln'.
    Determining projects to restore...
  All projects are up-to-date for restore.
  Project calculator is using configuration from 'C:\Source\husky-sandbox\calculator\obj\Debug\net6.0\calculator.GeneratedMSBuildEditorConfig.editorconfig'.
  Project calculator is using configuration from 'C:\Program Files\dotnet\sdk\6.0.400\Sdks\Microsoft.NET.Sdk\analyzers\build\config\analysislevel_6_default.editorconfig'.
  Project tests is using configuration from 'C:\Source\husky-sandbox\tests\obj\Debug\net6.0\tests.GeneratedMSBuildEditorConfig.editorconfig'.
  Project tests is using configuration from 'C:\Program Files\dotnet\sdk\6.0.400\Sdks\Microsoft.NET.Sdk\analyzers\build\config\analysislevel_6_default.editorconfig'.
  Running 2 analyzers on calculator.
  Running 2 analyzers on tests.
  Running 134 analyzers on calculator.
  Running 134 analyzers on tests.
  Formatted 0 of 10 files.
  Format complete in 5035ms.
  Determining projects to restore...
  All projects are up-to-date for restore.
  calculator -> C:\Source\husky-sandbox\calculator\bin\Debug\net6.0\calculator.dll
  tests -> C:\Source\husky-sandbox\tests\bin\Debug\net6.0\tests.dll
```

Running C# scripts is another cool feature of Husky.Net. Let's try to validate the commit message to follow a fixed pattern. Run `dotnet husky add commit-msg -c "echo 'running commit-msg'"`. Create a `.husky/commit-lint.csx` file with the following content:

```csharp
using System.Text.RegularExpressions;
using Internal;

private var pattern = @"^[aA-zZ]+-[\d]+(\s).+";
private var msg = File.ReadAllLines(Args[0])[0];

if (Regex.IsMatch(msg, pattern))
   return 0;

Console.WriteLine("Invalid commit message");
Console.WriteLine("e.g: ABC-123 brief description of the commit");

return 1;
``` 

Open the `.husky/commit-msg` file and add `husky exec .husky/commit-lint.csx --args "$1"` at the end. Run `git commit -m "wrong comment" --allow-empty`, to see at the end of the output something like this:

```
Invalid commit message
e.g: ABC-123 brief description of the commit
script execution failed
husky - commit-msg hook exited with code 1 (error)
``` 

The last example will run the tests before make the push to the remote repository. Run `dotnet husky add pre-push -c "echo 'running pre-push'"`. Open the file `.husky/pre-push` and add `dotnet test` at the end. Run `git commit -m "ABC-1234 initial commit" --allow-empty` and check the outputs. 

There are other features like the [Task Runner](https://alirezanet.github.io/Husky.Net/guide/task-runner.html), and maybe more in the future. We suggest keeping an eye on this project. All the code is available [here](https://github.com/raulnq/husky-sandbox). Thanks, and happy coding.