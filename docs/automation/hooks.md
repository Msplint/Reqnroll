# Hooks

Hooks (event bindings) can be used to perform additional automation logic at specific times, such as any setup required prior to executing a scenario. In order to use hooks, you need to add the `Binding` attribute to your class:

```{code-block} csharp
:caption: Hook File
[Binding]
public class MyHooks
{
    [BeforeScenario]
    public void SetupTestUsers()
    {
        //...
    }
}
```

Hooks are global, but can be restricted to run only for features or scenarios by defining a [scoped binding](scoped-bindings), which can be filtered with [tags](scoped-bindings.md#different-steps-for-different-tags). The execution order of hooks for the same type is undefined, unless [specified explicitly](#hook-execution-order).

The `[BeforeScenario]` in the following example will execute only for those scenarios that are (implicitly or explicitly) tagged with `@requiresUsers`.

```{code-block} csharp
:caption: Hook File
[Binding]
public class MyHooks
{
    [BeforeScenario("@requiresUsers")]
    public void SetupTestUsers()
    {
        //...
    }
}
```


## Supported Hook Attributes

| Attribute | Tag filtering* | Description |
|-----------|----------------|-------------|
| `[BeforeTestRun]`<br/>`[AfterTestRun]` | not possible | Automation logic that has to run before/after the entire test run (see note below). The method it is applied to must be static. |
| `[BeforeFeature]`<br/>`[AfterFeature]` | possible | Automation logic that has to run before/after executing each feature<br/> The method it is applied to must be static. |
| `[BeforeScenario]` or `[Before]`<br/>`[AfterScenario]` or `[After]` | possible | Automation logic that has to run before/after executing each scenario or scenario outline example |
| `[BeforeScenarioBlock]`<br/>`[AfterScenarioBlock]` | possible | Automation logic that has to run before/after executing each scenario block (e.g. between the "givens" and the "whens") |
| `[BeforeStep]`<br/>`[AfterStep]` | possible | Automation logic that has to run before/after executing each scenario step |

```{note}
As most of the unit test runners do not provide a hook for executing logic once the tests have been executed, the `[AfterTestRun]` event is triggered by the test assembly unload event. 

The exact timing and  thread of this execution may therefore differ for each test runner.
```

You can annotate a single method with multiple attributes.

## Using Hooks with Constructor Injection

You can use [context injection](context-injection) to access scenario level dependencies in your hook class using constructor injection.
For example you can get the `ScenarioContext` injected in the constructor:

```{code-block} csharp
:caption: Hook File
[Binding]
public class MyHooks
{
    private ScenarioContext _scenarioContext;

    public MyHooks(ScenarioContext scenarioContext)
    {
        _scenarioContext = scenarioContext;
    }

    [BeforeScenario]
    public void SetupTestUsers()
    {
        //_scenarioContext...
    }
}
```

```{note}
For static hook methods you can use parameter injection.
```

## Using Hooks with Parameter Injection

You can add parameters to your hook method that will be automatically injected by Reqnroll.
For example you can get the `ScenarioContext` injected as parameter in the BeforeScenario hook.

```{code-block} csharp
:caption: Hook File
[Binding]
public class MyHooks
{
    [BeforeScenario]
    public void SetupTestUsers(ScenarioContext scenarioContext)
    {
        //scenarioContext...
    }
}
```

Parameter injection is especially useful for hooks that must be implemented as static methods.

```{code-block} csharp
:caption: Hook File
[Binding]
public class Hooks
{
    [BeforeFeature]
    public static void SetupStuffForFeatures(FeatureContext featureContext)
    {
        Console.WriteLine("Starting " + featureContext.FeatureInfo.Title);
    }
}
```

In the BeforeTestRun hook you can resolve test thread specific or global services/dependencies as parameters.

```{code-block} csharp
:caption: Hook File
[BeforeTestRun]
public static void BeforeTestRunInjection(ITestRunnerManager testRunnerManager, ITestRunner testRunner)
{
    //All parameters are resolved from the test thread container automatically.
    //Since the global container is the base container of the test thread container, globally registered services can be also injected.

    //ITestRunManager from global container
    var location = testRunnerManager.TestAssembly.Location;
    
    //ITestRunner from test thread container
    var threadId = testRunner.ThreadId;
}
```

Depending on the type of the hook the parameters are resolved from a container with the corresponding lifecycle.

| Attribute | Container |
|-----------|-----------|
| `[BeforeTestRun]`<br/>`[AfterTestRun]` | TestThreadContainer |
| `[BeforeFeature]`<br/>`[AfterFeature]` | FeatureContainer    |
| `[BeforeScenario]`<br/>`[AfterScenario]`<br/>`[BeforeScenarioBlock]`<br/>`[AfterScenarioBlock]`<br/>`[BeforeStep]`<br/>`[AfterStep]`| ScenarioContainer |

{#hook-execution-order}
## Hook Execution Order

By default the hooks of the same type (e.g. two `[BeforeScenario]` hook) are executed in an unpredictable order. If you need to ensure a specific execution order, you can specify the `Order` property in the hook's attributes.

```{code-block} csharp
:caption: Hook File
[BeforeScenario(Order = 0)]
public void CleanDatabase()
{
    // we need to run this first...
}

[BeforeScenario(Order = 100)]
public void LoginUser()
{
    // ...so we can log in to a clean database
}
```

The number indicates the order, not the priority, i.e. the hook with the lowest number is always executed first.

If no order is specified, the default value is 10000. However, we do not recommend on relying on the value to order your tests and recommend specifying the order explicitly for each hook.

```{note}
If a hook throws an unhandled exception, subsequent hooks of the same type are not executed. If you want to ensure that all hooks of the same types are executed, you need to handle your exceptions manually.
```
```{note}
If a `BeforeScenario` throws an unhandled exception then all the scenario steps will be marked as skipped and the `ScenarioContext.ScenarioExecutionStatus` will be set to `TestError`.
```

## Tag Scoping

Most hooks support tag scoping. Use tag scoping to restrict hooks to only those features or scenarios that have *at least one* of the tags in the tag filter (tags are combined with OR). You can specify the tag in the attribute or using [scoped bindings](scoped-bindings).
