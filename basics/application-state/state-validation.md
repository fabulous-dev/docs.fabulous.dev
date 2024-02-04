# State validation

The model is the core data from which the whole state of the app can be resurrected. The model is generally immutable but may also contain elements such as service connections. It is common for the design of the model to grow “organically” as you prototype your app.

The init function returns your initial model. The update function updates the model as messages are received.

### Messages and Validation

Validation is generally done on updates to the model storing error messages from validation logic in the model so they can be correctly and simply displayed to the user. Here is a very basic example:

```fsharp
type Animal =
    | ValidAnimal of string
    | InvalidAnimal of string

type Model = { AnimalName : Animal }

type Msg = UpdateAnimal of string

let validAnimalNames = [ "Emu"; "Kangaroo"; "Platypus"; "Wombat" ]

let validateAnimal (animalName : string) =
    if List.contains animalName validAnimalNames
    then ValidAnimal animalName
    else InvalidAnimal animalName

let init () = { AnimalName = validateAnimal "Emu" }

let update msg model =
    match msg with
    | UpdateAnimal animalName -> { model with AnimalName = validateAnimal animalName }

let view (model: Model) : ViewElement =
    let makeEntryCell text =
        Entry(text, UpdateAnimal)
    
    ContentPage(
        "Animal",
        VStack() {
            match model.AnimalName with
            | ValidAnimal validName ->
                Entry(validName, UpdateAnimal)
            
            | InvalidAnimal invalidName ->
                Entry(invalidName, UpdateAnimal)
                Label($"{invalidName} is not a valid animal name. Try %A{validAnimalNames}") ]
        }
    )
```

A more advanced validation might use the `Result<'T,'TError>` type to wrap parts of the model that require validation: in the previous example the `Result` type has somewhat been reinvented. Using `Result` provides a consistent way of knowing which parts of the model are in a valid state, use of the standard `Result` functions like `map` and `bind` to perform branching logic, and more comprehensive error messaging. One thing to note is that `'TError` will usually need to carry the original input value so it can be displayed back to the user.

```fsharp
type Animal = Animal of string

type ErrorMessage =
    | InvalidName of InputString : string
    | BlankName

type Model =
    { AnimalName: string
      SelectedAnimal: Result<Animal, ErrorMessage> }

type Msg = UpdateName of string

let validAnimalNames = [ "Emu"; "Kangaroo"; "Platypus"; "Wombat" ]

let validateAnimal (animalName : string) =
    if animalName = ""
    then Error BlankName
    else
        if List.contains animalName validAnimalNames
        then Ok (Animal animalName)
        else Error (InvalidName animalName)

let init () =
    let name = "Emu" 
    { AnimalName = name
      SelectedAnimal = validateAnimal name }

let update msg model =
    match msg with
    | UpdateName name ->
        { model with
             AnimalName = name
             SelectedAnimal = validateAnimal animalName }

let view (model: Model) : ViewElement =
    ContentPage(
        "Animal",
        VStack() {
            Entry(model.AnimalName, UpdateName)
                
            match model.SelectedAnimal with
            | Ok (Animal animalName) ->
                Label($"You selected the animal {animalName}")
            | Error (InvalidName invalidName) ->
                Label($"{invalidName} is not a valid animal name. Try %A{validAnimalNames}")
            | BlankName ->
                Label("You must input a name")
        }
    )
```

Note that the same validation logic can be used in both your app and a service back-end.
