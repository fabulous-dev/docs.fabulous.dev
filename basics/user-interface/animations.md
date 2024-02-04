# Animations

Animations and focus are specified by accessing the underlying control and using the UI framework animation specifications. The underlying control is usually accessed via a `ViewRef`, akin to a `ref` in HTML/JavaScript and React.

* A `ViewRef` must have a sufficient scope that it lives long enough, e.g. a global scope or the scope of the model. The `ViewRef` can be held in the model itself if necessary.
* Initially `ViewRef` are empty. They will only be populated after the `view` function has been called and its results applied to the visual display.

For example, the following shows the creation of a `ViewRef` and associating it with a particular element:

```fsharp
let animatedLabelRef = ViewRef<Label>()

let view dispatch model = 
    Label("Rotate")
        .reference(animatedLabelRef) 
```

The underlying control can also be accessed by using the `created` handler:

```fsharp
let mutable label = None

View.Label(text="hello", created=(fun l ->  label <- Some l))
```

> NOTE: A `ViewRef` only holds a weak handle to the underlying control.\
> The `Value` property may thus fail if the underlying control has been collected.\
> As a result it is often sensible to use the `TryValue` property which returns an option.

### Animations

Animations are specified by using an animation specification on the underlying control, e.g.

```fsharp
let animatedLabelRef = ViewRef<Label>()

let update msg model =
    match msg with 
    | Poked ->
        match animatedLabelRef.TryValue with 
        | None -> () 
        | Some c -> c.RotateTo (360.0, 2000u) |> ignore

let view dispatch = 
    VStack() {
        Label("Rotate")
            .reference(animatedLabelRef)
            
        Button("Rotate", Poked)
    }
```

Animations in specify tasks. These are ignorable if the animation is simple. Composite animations must compose tasks, either by using `async { ...}` and `Async.AwaitTask` and `Async.StartAsTask`, or by using `task { ... }` from the F# community `TaskBuilder` library.

Examples of custom tasks are shown in C# syntax in [Animation in Xamarin.Forms](https://docs.microsoft.com/en-us/xamarin/xamarin-forms/user-interface/animation/).
