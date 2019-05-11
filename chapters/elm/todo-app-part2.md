# To-Do List Application: Part 1

In this section, we will continue to build upon what we did in the previous section and upgrade our To-Do list application such that it is able to mark To-Do items as completed as well as the ability to delete them from the list. It will look like the following:

<div style="width:100%">
  <div style="margin: 0 auto; width:65%;"> 
    <resolved-image source="/images/elm/todo-part2.gif" />      
  </div>
</div>

You can see and use the application [live here](https://zaid-ajaj.github.io/elmish-todo-part2/).

### Modelling the State

Previously, when modelling the To-Do items in our state, we chose to use `string list` because that is what they were. However, now these items have to carry more information, namely whether they are *completed* or not. So one might consider using a separate type for the To-Do item as follows:
```fsharp {highlight: ['1-4', 8]}
type Todo = {
  Description: string
  Completed: bool
}

type State = {
  NewTodo: string
  TodoList : Todo list 
}
```
Alright, from the point of view of the `State` this looks like enough information. But we are indeed missing a very important piece: the ability to *identify* the individual To-Do items. This becomes clear when we try to encode our events in the `Msg` type:
```fsharp
type Msg = 
  | SetNewTodo of string
  | AddTodo
  | ToggleCompleted of ???
  | DeleteTodo of ??? 
```
Here I have added two events: `ToggleCompleted` and `DeleteTodo`, these events are triggered when I want to toggle the completed flag or delete a *certain* To-Do item, but how do I know which one to toggle or to delete? The answer is to extend our state type and add an identity field to each of To-Do items:
```fsharp {highlight: [2]}
type Todo = {
  Id : int 
  Description: string
  Completed : bool 
}
```
Now the events can carry information about a certain To-Do item only by using the identity associated with that item:
```fsharp
type Msg = 
  | SetNewTodo of string
  | AddTodo
  | ToggleCompleted of int
  | DeleteTodo of int 
```
For example, when an event `ToggleCompleted 4` it means "Toggle the completed flag of the item with identity = 4" and the same holds for `DeleteTodo 4` which means "Delete the item that has identity = 4". Let's try to imagine how the state evolves as these events are triggered:
```bash
-> Initial State
  { 
    NewTodo = ""; 
    TodoList = [ 
      { Id = 1; Description = "Learn F#"; Completed = true }
      { Id = 2; Description = "Learn Elmish"; Completed = false } 
    ] 
  } 

-> Trigger [ToggleCompleted 2]
-> Next State =  
 { 
    NewTodo = ""; 
    TodoList = [ 
      { Id = 1; Description = "Learn F#"; Completed = true }
      { Id = 2; Description = "Learn Elmish"; Completed = true } 
    ] 
  } 

-> Trigger [DeleteTodo 1]
-> Next State = 
  { 
    NewTodo = ""; 
    TodoList = [ 
      { Id = 2; Description = "Learn Elmish"; Completed = true } 
    ] 
  } 

-> Trigger [SetNewTodo "Have fun and profit"]
-> Next State = 
  { 
    NewTodo = "Have fun and profit"; 
    TodoList = [ 
      { Id = 2; Description = "Learn Elmish"; Completed = true } 
    ] 
  } 

-> Trigger [AddTodo]
-> Next State = 
  { 
    NewTodo = ""; 
    TodoList = [ 
      { Id = 2; Description = "Learn Elmish"; Completed = true } 
      { Id = 3; Description = "Have fun and profit"; Completed = false } 
    ] 
  } 
```
Notice how when we are adding a new todo, we are incrementing the `Id` by 1 based on the maximum value of `Id` in the list. Also we initialize the `Completed` flag of a new `Todo` item to be false. 

Using the steps above, we have a rough idea of how we should implement the `update` function, let's try to implement it concretely, and go through the code:
```fsharp
let update msg state = 
  match msg with 
  | SetNewTodo desc -> 
      { state with NewTodoDescription = desc }

  | DeleteTodo todoId ->
      let nextTodoList = 
        state.TodoList
        |> List.filter (fun todo -> todo.Id <> todoId)
      
      { state with TodoList = nextTodoList }    

  | ToggleCompleted todoId ->
      let nextTodoList = 
        state.TodoList
        |> List.map (fun todo -> 
           if todo.Id = todoId 
           then { todo with Completed = not todo.Completed }
           else todo)
 
      { state with TodoList = nextTodoList }

  | AddNewTodo when state.NewTodoDescription = "" ->
      state 
  
  | AddTodo ->
      let nextTodoId = 
        match state.TodoList with
        | [ ] -> 1
        | elems -> 
            elems
            |> List.maxBy (fun todo -> todo.Id)  
            |> fun todo -> todo.Id + 1

      let nextTodo = 
        { Id = nextTodoId
          Description = state.NewTodoDescription
          Completed = false }
          
      { state with 
          NewTodoDescription = ""
          TodoList = List.append state.TodoList [nextTodo] }
```
Let's go through each event: `DeleteTodo`
```fsharp
| DeleteTodo todoId ->
    let nextTodoList = 
      state.TodoList
      |> List.filter (fun todo -> todo.Id <> todoId)
    
    { state with TodoList = nextTodoList }  
```
Here "deleting" a To-Do item is a matter of creating a *new* list where the To-Do item to be deleted is filtered out. This is common in Elmish, the state is immutable we create new states rather than mutating the current one.

As for `ToggleCompleted`:
```fsharp
| ToggleCompleted todoId ->
    let nextTodoList = 
      state.TodoList
      |> List.map (fun todo -> 
         if todo.Id = todoId 
         then { todo with Completed = not todo.Completed }
         else todo)

    { state with TodoList = nextTodoList }
```
We transform (map) each item in the list of our To-Do items and we check: if `todo` has the id of one we want to toggle, then we return a *new* To-Do item where the `Completed` field is toggled. Otherwise, just return the To-Do item as is unchanged. That's we get a new list where one To-Do item is toggled. Next we return a new state with the new list we just created. 

Event `AddTodo` now has a bit more logic to it than from the previous section:
```fsharp
| AddTodo ->
    let nextTodoId = 
      match state.TodoList with
      | [ ] -> 1
      | elems -> 
          elems
          |> List.maxBy (fun todo -> todo.Id)  
          |> fun todo -> todo.Id + 1

    let nextTodo = 
      { Id = nextTodoId
        Description = state.NewTodo
        Completed = false }
        
    { state with 
        NewTodoDescription = ""
        TodoList = List.append state.TodoList [nextTodo] }
```
First we calculate the identity that our next To-Do item will have, we do so by checking the current list of `Todo`'s, if the list is empty, then use 1 is the indentity for the first item, otherwise we get the To-Do item that has the largest `Id` value using `List.maxBy` and we extract the `Id` from that item. Afterwards we create a new `Todo` using the `Id` we calculated and adding (appending) it the `TodoList` we already have in the state. 

### The User Interface

That was it for the `update`, now we consider the `render` function. Since the user interface is more or less the same as the in the previous section, `render` will look almost the same, except now we have more logic when rendering the individual To-Do items. Because the code in `render` is starting to explode, I will be breaking parts of it into separate functions: 
```fsharp {highlight: [10, 12]}
let createTodoTextbox (state: State) (dispatch: Msg -> unit) = 
  (* render the text box and the Add To-Do button *)

let renderTodo (todo: Todo) (dispatch: Msg -> unit) = 
  (* render a single To-Do item *)

let render (state: State) (dispatch: Msg -> unit) =
  div [ Style [ Padding 20 ] ] [
    h3 [ Class "title" ] [ str "Elmish To-Do list" ]
    createTodoTextbox state dispatch
    div [ Class "content"; Style [ MarginTop 20 ] ] [ 
      for todo in state.TodoList -> renderTodo todo dispatch
    ]
  ]  
```