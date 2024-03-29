# Error monitoring

In Fabulous, everything happens in a centralized message loop that is responsible for calling your `init`, `update` and `view` functions.\
Fabulous allows you to plug into this loop to run custom logic such as logging and error handling.

There’s a few built-in functions available already:

| Functions                    | Description                                                                                                                                                                                        |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Program.withTrace            | <p>Call custom tracing function everytime Fabulous needs to update the app.<br>Signature: <code>trace: 'msg -> 'model -> unit</code></p>                                                           |
| Program.withExceptionHandler | <p>Call custom error handling logic. Returns a boolean to indicate if the exception has been handled or not.<br>Signature: <code>handler: exn -> bool</code></p>                                   |
| Program.withLogger           | <p>Fabulous has a lot of internal logs to help debugging what is happening. By default, the logs won't be printed anywhere but it can be configured.<br>Signature: <code>logger: Logger</code></p> |

F#To use them, you will need to add them when declaring the runner.\
You can add multiple trace functions, one after another.

```fsharp
module App =
    let runner = 
        Program.stateful init update view
        |> Program.withConsoleTrace
        |> Program.withErrorHandler (fun (message, exn) -> writeToDisk exn)
```

In this example, logs will be written to the console before the error handler can write the exceptions to the disk.

### Writing a custom trace function&#x20;

Writing a custom trace function is simple.\
You need to make a function that accepts a `Program<'args, 'model, 'msg, 'view>` and that outputs another `Program<'args, 'model, 'msg, 'view>`.

This `Program` defines the handlers that are responsibles for calling `init`, `update` and `views` as well as handle errors. You can define your own handlers instead to do additional logic.\
**Make sure to call the previous `Program` handlers in your owns, otherwise you will completely bypass Fabulous.**

Here’s a simple example that prints a message each time something happens

```fsharp
let withSimpleTrace (program: Program<'args, 'model, 'msg, _>) =
    let traceInit args =
        Console.WriteLine "Init"
        program.init args

    let traceUpdate msg model =
        Console.WriteLine "Update"
        program.update msg model

    let traceView model dispatch =
        Console.WriteLine "View"
        program.update msg model

    let traceOnError (message, exn) =
        Console.WriteLine "Error"
        program.onError (message, exn)
            
    { program with
        init = traceInit 
        update = traceUpdate
        view = traceView
        onError = traceOnError }

module App =
    let runner = 
        Program.stateful init update view
        |> Program.withSimpleTrace
```

You’re not required to implement all handlers, if you only need to override `update` then just override that one.

```fsharp
let withSimpleTrace (program: Program<'args, 'model, 'msg, _>) =
    let traceUpdate msg model =
        Console.WriteLine "Update"
        program.update msg model

    { program with update = traceUpdate }
```

### AppCenter&#x20;

Here’s a good example of implementing our own trace function.

[Visual Studio App Center](https://appcenter.ms/) is a DevOps portal tailored for mobile application development.\
It handles everything from build, tests, distribution, analytics and crashes reporting, for Android, iOS and UWP.

We can define our own trace functions to send analytics and crashes to AppCenter.

AppCenter provides us with a really simple way to report to it.\
We only need to install the following packages:

* [`Microsoft.AppCenter.Analytics`](https://www.nuget.org/packages/Microsoft.AppCenter.Analytics/)
* [`Microsoft.AppCenter.Crashes`](https://www.nuget.org/packages/Microsoft.AppCenter.Crashes/)

Once the packages added, we can access 3 methods:

* `Start`: Initialize AppCenter by providing our app secrets
* `Analytics.TrackEvent`: Track a custom event with associated data
* `Crashes.TrackError`: Track an exception.

AppCenter will provide a dashboard of those events and exceptions along with stack traces

Here we override `update` to call `Analytics.TrackEvent`, and `onError` to call `Crashes.TrackError`.

```fsharp
module AppCenter =
    type AppCenterUpdateTracer<'msg, 'model> =
        'msg -> 'model -> (string * (string * string) list) option

    /// Trace all the updates to AppCenter
    let withAppCenterTrace (shouldTraceUpdate: AppCenterUpdateTracer<_, _>) (program: Program<_, F#_, _, _>) =
        let traceUpdate msg model =
            match shouldTraceUpdate msg model with
            | Some (key, value) -> Analytics.TrackEvent (key, dict value)
            | None -> ()
            program.update msg model

        let traceError (message, exn) =
            Crashes.TrackError(exn, dict [ ("Message", message) ])

        { program with
            update = traceUpdate 
            onError = traceError }
```

We could trace everything, but you should consider to trace the minimum to protect your users' privacy.\
To do that, we have added a `AppCenterUpdateTracer` function that will filter the messages that interest us and what data we should extract from it.

```fsharp
module Tracing =
    let hasValue = (not << String.IsNullOrEmpty) >> string

    let rules msg _ =
        match msg with
        | App.Msg.GoToAbout ->
            Some ("Navigation", [ ("Page", "About") ])
        | App.Msg.NavigationPopped ->
            Some ("Back Navigation", [])
        | App.Msg.UpdateWhenContactAdded c ->
            Some ("Contact added", [
                ("Has Email", hasValue c.Email)
                ("Has Phone", hasValue c.Phone)
                ("Has Address", hasValue c.Address)
            ])
        | _ -> None
```

In our `App` class, we need to call `AppCenter.Start("appsecrets", typeof<Analytics>, typeof<Crashes>)` to initialize it.

```fsharp
module App =
    do AppCenter.Start("ios=(...);android=(...)", typeof<Analytics>, typeof<Crashes>)

    let runner = 
        Program.stateful init update view
        |> AppCenter.withAppCenterTrace Tracing.rules
```
