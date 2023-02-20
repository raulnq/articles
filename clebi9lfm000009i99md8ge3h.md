# Mutation Testing with Stryker.NET

How do we know that the unit tests we develop are efficient enough to ensure the quality of our code? We usually believe that high code coverage is a good indicator that our code does what we expect, but this is not always true. Let's test this with an example, execute the following commands:

```bash
dotnet new classlib -n calculator
dotnet new mstest -n tests
dotnet new sln -n MutationTesting
dotnet sln add calculator
dotnet sln add tests
dotnet add tests/tests.csproj reference calculator/calculator.csproj
```

Open the solution and create the `Calculator.cs` file with the following code:

```csharp
public class Calculator
{
    public decimal Multiply(decimal a, decimal b)
    {
        return a * b;
    }

    public decimal Divide(decimal a, decimal b)
    {
        return a / b;
    }

    public decimal Add(decimal a, decimal b)
    {
        return a + b;
    }

    public decimal Subtrac(decimal a, decimal b)
    {
        return a - b;
    }
}
```

Then create the `Tests.cs` file with the worst/laziest test cases we can imagine:

```csharp
[TestClass]
public class Tests
{
    [TestMethod]
    public void one_multiplied_by_one_should_be_one()
    {
        var sut = new Calculator();

        var r = sut.Multiply(1, 1);

        Assert.AreEqual(1, r);
    }

    [TestMethod]
    public void one_divided_by_one_should_be_one()
    {
        var sut = new Calculator();

        var r = sut.Divide(1, 1);

        Assert.AreEqual(1, r);
    }


    [TestMethod]
    public void zero_plus_zero_should_be_zero()
    {
        var sut = new Calculator();

        var r = sut.Add(0, 0);

        Assert.AreEqual(0, r);
    }

    [TestMethod]
    public void zero_minus_zero_should_be_zero()
    {
        var sut = new Calculator();

        var r = sut.Subtrac(0, 0);

        Assert.AreEqual(0, r);
    }
}
```

Let's find out the code coverage of our tests with [Coverlet](https://github.com/coverlet-coverage/coverlet). Run `dotnet build` and then `coverlet ./tests/bin/Debug/net7.0/tests.dll --target "dotnet" --targetargs "test --no-build --no-restore"`. The output will be something like this:

```bash
coverlet ./tests/bin/Debug/net7.0/tests.dll --target "dotnet" --targetargs "test --no-build --no-restore"
Test run for D:\Source\Github\mutation-testing-sandbox\tests\bin\Debug\net7.0\tests.dll (.NETCoreApp,Version=v7.0)
Microsoft (R) Test Execution Command Line Tool Version 17.4.0 (x64)
Copyright (c) Microsoft Corporation.  All rights reserved.
Starting test execution, please wait...
A total of 1 test files matched the specified pattern.
Passed!  - Failed:     0, Passed:     4, Skipped:     0, Total:     4, Duration: 18 ms - tests.dll (net7.0)

Calculating coverage result...
  Generating report 'D:\Source\Github\mutation-testing-sandbox\coverage.json'
+------------+------+--------+--------+
| Module     | Line | Branch | Method |
+------------+------+--------+--------+
| calculator | 100% | 100%   | 100%   |
+------------+------+--------+--------+

+---------+------+--------+--------+
|         | Line | Branch | Method |
+---------+------+--------+--------+
| Total   | 100% | 100%   | 100%   |
+---------+------+--------+--------+
| Average | 100% | 100%   | 100%   |
+---------+------+--------+--------+
```

Cool, 100% percent of coverage. Sadly, there is another implementation that could pass all the tests:

```csharp
public class Calculator
{
    public decimal Multiply(decimal a, decimal b)
    {
        return 1;
    }

    public decimal Divide(decimal a, decimal b)
    {
        return 1;
    }

    public decimal Add(decimal a, decimal b)
    {
        return 0;
    }

    public decimal Subtrac(decimal a, decimal b)
    {
        return 0;
    }
}
```

So it looks like we are doing something wrong. Luckily **Mutation Testing** can help here:

> *Mutation testing is a fault-based testing technique where variations of a software program are subjected to the test dataset. This is done to determine the effectiveness of the test set in isolating the deviations.*

During the Mutation Testing, the following steps are executed:

* Faults are introduced into the code by creating many versions called **mutants**. Each mutant should contain a single fault, and the goal is to cause the mutant version to fail which demonstrates the effectiveness of the test cases.
    
* Test cases are applied to the original program and also to the mutant program.
    
* Compare the results of an original and mutant program.
    
* If the original program and mutant programs generate different outputs, then the mutant is killed by the test case. Hence the test case is good enough to detect the change between the original and the mutant program.
    
* If the original and mutant programs generate the same output, the mutant is kept alive. In such cases, more effective test cases need to be created that kill all mutants.
    

In the .NET ecosystem, we have [Stryker.NET](http://Stryker.NET):

> [Stryker.NET](http://Stryker.NET) offers mutation testing for your .NET Core and .NET Framework projects. It allows you to test your tests by temporarily inserting bugs.

Run the `dotnet tool install -g dotnet-stryker` command to install Stryker.NET and then `dotnet stryker`, the following output will be shown:

```bash
Version: 3.6.1

[07:50:13 INF] Identifying projects to mutate in D:\Source\Github\mutation-testing-sandbox\MutationTesting.sln. This can take a while.
[07:50:15 INF] Found 1 source projects
[07:50:15 INF] Found 1 test projects
[07:50:20 INF] The project D:\Source\Github\mutation-testing-sandbox\calculator\calculator.csproj will be mutated.
[07:50:20 INF] Analysis complete.
[07:50:22 INF] Total number of tests found: 4.
[07:50:22 INF] Initial testrun started.
[07:50:26 INF] 8 mutants created
[07:50:26 INF] Capture mutant coverage using 'CoverageBasedTest' mode.
[07:50:27 INF] 8     total mutants will be tested
������������������������������������������������������������������������������������������������������������������������
�����������������������������������������������������������������������������������������������������������������������0
�����������������������������������������������������������������������������������������������������������������������0
�100.00% � Testing mutant 8 / 8 � K 2 � S 6 � T 0 � ~0m 00s �                                                    00:00:04

Killed:   2
Survived: 6
Timeout:  0
Hint: by passing "--open-report or -o" the report will open automatically once Stryker is done.

Your html report has been generated at:
D:\Source\Github\mutation-testing-sandbox\StrykerOutput\2023-02-19.07-50-13\reports\mutation-report.html
You can open it in your browser of choice.
[07:50:31 INF] Time Elapsed 00:00:18.2671985
[07:50:31 INF] The final mutation score is 25.00 %
```

Open the generated report to see a summary of the execution:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1676811380294/83430ada-5316-402e-8bb2-d0a655688bf0.png align="center")

Stryker.NET generated eight mutants and our tests only killed two of them. Click the first row of the report to see the mutants created ([here](https://stryker-mutator.io/docs/stryker-net/mutations/) you can find all the mutators provided by Stryker.NET):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1676811780548/6bb1f9b8-d335-49ca-9e99-3341180c1efe.png align="center")

In this case, Stryker.NET only changed the `+` operator to the `-` operator. Based on those findings we could enhance our tests as follows:

```csharp
[TestClass]
public class Tests
{
    [TestMethod]
    [DataRow(1, 1, 1)]
    [DataRow(2, 3, 6)]
    [DataRow(7, 5, 35)]
    public void x_multiplied_by_y_should_be_z(int x, int y, int z)
    {
        var sut = new Calculator();

        var r = sut.Multiply(x, y);

        Assert.AreEqual(z, r);
    }

    [TestMethod]
    [DataRow(1, 1, 1)]
    [DataRow(10, 5, 2)]
    [DataRow(24, 3, 8)]
    public void x_divided_by_y_should_be_z(int x, int y, int z)
    {
        var sut = new Calculator();

        var r = sut.Divide(x, y);

        Assert.AreEqual(z, r);
    }


    [TestMethod]
    [DataRow(0, 0, 0)]
    [DataRow(5, 3, 8)]
    [DataRow(2, 9, 11)]
    public void x_plus_y_should_be_z(int x, int y, int z)
    {
        var sut = new Calculator();

        var r = sut.Add(x, y);

        Assert.AreEqual(z, r);
    }

    [TestMethod]
    [DataRow(0, 0, 0)]
    [DataRow(5, 1, 4)]
    [DataRow(11, 5, 6)]
    public void x_minus_y_should_be_z(int x, int y, int z)
    {
        var sut = new Calculator();

        var r = sut.Subtrac(x, y);

        Assert.AreEqual(z, r);
    }
}
```

Run Stryker.NET again:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1676812387597/8ffe7e4c-67f2-4c2e-9d86-7b5f8da82bb3.png align="center")

This time our test killed all the mutants, giving us more confidence in the tests we wrote. Mutation testing is gaining more adepts those days, but the computational cost could be huge under programs with thousands of lines. Remember that we run all our tests against one mutant (our code with only one change). That means a huge program could generate hundreds of mutants, and testing them all could take a long time. So, the advice is to use this technique only under critical code (code related to a business feature with a high value). Thank you, and happy coding.