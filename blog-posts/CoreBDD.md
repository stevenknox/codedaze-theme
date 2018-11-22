## What is CoreBDD?


CoreBDD is a lightweight open-source [Behaviour-Driven Development](https://en.wikipedia.org/wiki/Behavior-driven_development) framework built on-top of the popular [xUnit.net](https://xunit.github.io/) testing tool that enables tests using the Gherkin language. [All code is available on GitHub](https://github.com/stevenknox/CoreBDD) and packages ready to use in your project [can be found on NuGet](https://www.nuget.org/packages?q=corebdd)

For those unfamilar with 'Gherkin', it is a [Business Readable, Domain Specific Language](http://martinfowler.com/bliki/BusinessReadableDSL.html) created especially for behaviour descriptions. It gives you the ability to remove logic details from behavior tests. A Gherkin specification is composed of 'Steps' that begin with a specifc keyword (When, Then, And, But etc). An example specification defining the use of a calculator could look like

  ```` gherkin
Feature: Calculator
  In order to avoid silly mistakes
  As a math idiot
  I want to be told the sum of two numbers

  Scenario: Add two numbers
  			Given I have entered 1 into the calculator
  			And I have also entered 2 into the calculator
  			When I press add
  			Then the result should be 3

  Scenario: Subtract two numbers
  			Given I have entered 5 into the calculator
  			And I have also entered 2 into the calculator
  			When I press minus
  			Then the result should be 3
````

CoreBDD allows for cucumber style tests with a class representing a scenario and a method as a step definition. There is also the ability to contain a scenario into a single method using a delegate for each step. CoreBDD maps xUnit concepts to Gherkin language to ensure your tests closely follow the language of a BDD specification. The table below shows an example of some of the mappings


|xUnit Attribute| CoreBDD Attribute  |
|--|--|
| Fact | Scenario |
| Theory | ScenarioOutline |
| InlineData | Examples |


CoreBDD provides an extension library for Spec Generation that leverages the power of the Roslyn compiler to generate Gherkin feature files directly from your code, and also the ability to generate test classes from an existing Gherkin feature file.

The command line tool provides developers with easy access to these code generation features and samples of how to use the tool are at the bottom of this post.

## Get Started

There are several ways to get started with CoreBDD. You can install the dotnet project template via

````ruby
    dotnet new -i corebdd.projecttemplate
````

Then create a new folder for your test project and run

````ruby
    dotnet new corebdd
````

Alternatively you can add CoreBDD to an existing xUnit test project via the nuget package

````ruby
    dotnet add package CoreBDD
````

There is also an optional Command Line tool for running tests with custom output, scaffolding test classes (features/scenarios/steps) and two-way code generation (Gherkin to CoreBDD tests and vice-versa). More documentation on the CLI is available at the bottom of this page.

````ruby
   dotnet tool install -g corebdd.commandline
````

Finally if you are using VSCode I have made some code-snippets available
[available here](https://gist.github.com/stevenknox/231f1dcf775ceeeb890edc634ad551a2)

## Writing CoreBDD Tests

Following the usual calculator example, we can start with the following model to test
 
   
  ````csharp
public class Calculator
    {
        public int Add(int x, int y) => x + y;
        public int Subtract(int x, int y) => x - y;
    }
````

We can define a *Feature* to collate a suite of scenarios by deriving from the *Specification* base class and decorating with the *Feature* attribute. Note both constructors are required to support the different test syntax styles.

  ````csharp
    [Feature("Calculator", 
    @"In order to avoid silly mistakes
    As a math idiot
    I want to be told the sum of two numbers")]
    public class CalculatorFeature : Specification
    {
         public CalculatorFeature(SpecFixture context):base(context) 
         {
            
         }
         
         public CalculatorFeature()
         {
            
         }
    }
````
Once we have created our base Feature, we have several different flavours for writing tests, first we can generate a scenario-per-class (similar to Cucumber style tests) with a method for each Given/When/Then step. To do this simply inherit from the new Feature class, decorate with an Example attribute and provide Given, When, Then methods that will execute in order

  ````csharp
    [Example("Add two numbers")]
    public class AddTwoNumbers : CalculatorFeature
    {
        readonly Calculator calc;

        public AddTwoNumbers(GivenWhenThenFixture state) 
            : base(state) => calc = new Calculator();

        [Given("I have entered {0} into the calculator", 1)]
        public void Given(int first) => Context.Given.First = first;

        [And("I have also entered {0} into the calculator", 2)]
        public void And(int second) => Context.Given.Second = second;

        [When("I press add")]
        public void When() => Context.When = calc.Add(Context.Given.First, Context.Given.Second);

        [Then("the result should be {0}", 3)]
        public void Then(int result) => Context.Result.ShouldBe(result);
    }
````


You can also define scenarios in a single method using delgates for each of the steps and allowing for multiple scenarios to be defined within the same class

  ````csharp
    public class AdvancedCalculator : CalculatorFeature
    {
        Calculator calculator;

        [Scenario("Multiply two numbers")]
        public void MultiplyTwoNumbers()
        {
            Given("I have a calculator",           () => calculator = new Calculator());
            When("I key in 10",                    () => calculator.Key(10));
            And("I key in 5 and press multiply",  () => calculator.Multiply(5));
            Then("It sets the Total to 50",        () => calculator.Total.ShouldBe(50));
            And("It sets the equation to 10 x 5", () => calculator.Equation.ShouldBe("10 x 5"));
        }

        [Scenario("Divide two numbers")]
        public void DivideTwoNumbers()
        {
            Given("I have a calculator",       () => calculator = new Calculator());
            When("I key in 42",                () => calculator.Key(42));
            Then("It sets the Total to 42",    () => calculator.Total.ShouldBe(42));
            And("It sets the equation to 42", () => calculator.Equation.ShouldBe("42"));
        }
    }   
````

The method based syntax also supports data driven tests, using xUnit InlineData (class based scenarios don't support data driven tests just yet).

  ````csharp
        [ScenarioOutline("Divide two numbers")]
        [Examples(10, 2, 5)]
        [Examples(20, 4, 5)]
        public void DivideTwoNumbers(int number, int divideby, int result)
        {
            Given($"I have a calculator",                           () => calculator = new Calculator());
            When($"I key in {number}",                              () => calculator.Key(number));
            And($"I key in {divideby} and press divide",           () => calculator.Divide(divideby));
            Then($"It sets the Total to {result}",                  () => calculator.Total.ShouldBe(result));
            And($"It sets the equation to {number} / {divideby}",  () => calculator.Equation.ShouldBe($"{number} / {divideby}"));
        }
````

You can generate Gherkin specs from your tests using the *CoreBDD.SpecGeneration* extension library, either by calling from an application or command line tool and passing in the path to the assembly containing tests, or by hooking up your test project to generate the specs after the test run. 

To do the latter, first reference the *CoreBDD.SpecGeneration* library

 ````ruby
    dotnet add package CoreBDD.SpecGeneration
````

Next create a Fixture class within your test project, and call *GenerateSpecs.OutputFeatureSpecs* within the Dispose method, passing in the Assembly (or path to the Assembly) and the output folder for the generated specs.


  ````csharp
    [CollectionDefinition("CoreBDD")]
    public class Collection : ICollectionFixture<GenerateSpecsFixture> { }

    public class GenerateSpecsFixture : IDisposable
    {
        public void Dispose()
        {
            GenerateSpecs.OutputFeatureSpecs(this.GetType().Assembly.Location, @"..\..\..\Specs\");
        }
    }
````

When the tests complete running, a *FeatureName.feature* file is generated under the Specs folder of the xUnit test project. It generates Gherkin specs for the feature and related scenarios. Example *CalculatorFeature.feature* :

  ````gherkin
    Feature: Calculator
	In order to avoid silly mistakes
    As a math idiot
    I want to be told the sum of two numbers

    Scenario: Add two numbers
    			Given I have entered 1 into the calculator
    			And I have also entered 2 into the calculator
    			When I press add
    			Then the result should be 3

    Scenario: Subtract two numbers
    			Given I have entered 5 into the calculator
    			And I have also entered 2 into the calculator
    			When I press minus
    			Then the result should be 3

````

## Command Line Tool

The command line tool makes it easy to run tasks such as test execution with Gherkin style output, generating default feature and scenario test files and generating Gherkin feature files from existing tests, or generating tests from existing feature files.

Starting from scratch using the dotnet template and cli tools:

````ruby
    mkdir demobdd
    cd demobdd
````
Next create the new CoreBDD project

````ruby
    dotnet new corebdd
````

Find CoreBDD tests in current and sub directories and execute tests

````ruby
    corebdd test
````

Run tests then generate Gherkin .feature files in specified location

````ruby
    corebdd test --specs --output ./Specs
````

Scaffold a CoreBDD feature class called 'Login' in current folder

````ruby
    corebdd generate feature --name login --namespace demobdd
````

Scaffold a CoreBDD scenario class called 'LoginToWebsite' under the 'Login' feature

````ruby
corebdd generate scenario --name LoginToWebsite --feature login --namespace demobdd
````

Scaffold CoreBDD Tests from existing gherkin '.feature' files, specifying location of feature files and target folder for generated tests.
If you have been following using the 'corebdd' test example, delete the 'Features' folder (leaving the Specs folder with .feature files intact) then run:

````ruby
corebdd generate tests --path ./Specs --output ./Features --namespace demobdd
````

You should now have test stubs regenerated using the .feature file scenarios.

## Awesome .NET Core

I'm delighted the project has been recently added to the [Awesome .NET Core list on GitHub](https://github.com/thangchung/awesome-dotnet-core) and hope this will drive adoption and maybe even help build some community around the project. If you are interested and can contribute by adding some features, fixing bugs or even refactoring some existing content I would love to hear from you or receive a pull-request!