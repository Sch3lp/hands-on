# Building the frontend

To render and manage user interface elements, we will use the popular framework [React](https://reactjs.org/) together with the available [wrappers](https://github.com/JetBrains/kotlin-wrappers/) for Kotlin. It may feel like using a web framework would be too much for a small project like this - but setting up a full project with React allows you to re-use this project and its configuration as a jumping-off point for future multiplatform applications that might have higher complexity.

Building web applications with React can be a complex topic in itself, so wherever possible, explanations and details relating to the topic are kept light. If you'd like to get a more in-depth view of typical workflows and how apps are developed with React and Kotlin/JS, please refer to the [Building Web Applications with React and Kotlin/JS](https://play.kotlinlang.org/hands-on/Building%20Web%20Applications%20with%20React%20and%20Kotlin%20JS/01_Introduction) hands-on.

First up, we first need to obtain some data from the server to display. For this, we'll build a small API client.

### Writing our API client

Our API client will use the [`ktor-clients`](https://ktor.io/clients/index.html) library to send requests to our HTTP endpoints. Ktor clients use Kotlin's coroutines to provide non-blocking networking – and just like the Ktor server, the clients support *Plugins*. One feature we will make use of is `JsonFeature`.

In our configuration, the `JsonFeature` makes use of `kotlinx.serialization` to provide us with a way to create typesafe HTTP requests, by taking care of the automatic conversion between Kotlin objects to their JSON representation and vice versa.

Leveraging these properties, we can create our API wrapper as a set of suspending functions that either accept or return `ShoppingItems`. We implement them as follows in `src/jsMain/kotlin`, in a file called `Api.kt`:

```kotlin
import io.ktor.http.*
import io.ktor.client.*
import io.ktor.client.request.*
import io.ktor.client.features.json.JsonFeature
import io.ktor.client.features.json.serializer.KotlinxSerializer

import kotlinx.browser.window

val endpoint = window.location.origin // only needed until https://youtrack.jetbrains.com/issue/KTOR-453 is resolved

val jsonClient = HttpClient {
    install(JsonFeature) { serializer = KotlinxSerializer() }
}

suspend fun getShoppingList(): List<ShoppingListItem> {
    return jsonClient.get(endpoint + ShoppingListItem.path)
}

suspend fun addShoppingListItem(shoppingListItem: ShoppingListItem) {
    jsonClient.post<Unit>(endpoint + ShoppingListItem.path) {
        contentType(ContentType.Application.Json)
        body = shoppingListItem
    }
}

suspend fun deleteShoppingListItem(shoppingListItem: ShoppingListItem) {
    jsonClient.delete<Unit>(endpoint + ShoppingListItem.path + "/${shoppingListItem.id}")
}
```

Next, let's use these functions to get a list on the screen!

### Building the user interface

We've laid the groundwork on the client, and have a clean API to access the data that's being provided by the server. We can now work on displaying this data in a React application.

#### Configuring an entry point for our application

Instead of rendering a simple "Hello, Kotlin/JS" string, it's time we make our application render a functional `App` component.

Replace the content inside `src/jsMain/kotlin/Main.kt` with the following:

```kotlin
import react.dom.render
import kotlinx.browser.document
import react.create

fun main() {
    val container = document.getElementById("root") ?: error("Couldn't find container!")
    val root = createRoot(container)
    root.render(App.create())
}
```

#### Building and rendering the shopping list

Next up is the implementation for the `App` component we would like to render. For our shopping list application, we need to

- keep the "local state" of the shopping list (which elements to display),
- load the shopping list elements from the server (and set the state accordingly), and
- provide React with instructions on how to render the list.

Based on these requirements, we can implement the `App` component as follows. We create and fill the file `src/jsMain/kotlin/App.kt`:

```kotlin
import react.*
import kotlinx.coroutines.*
import react.dom.html.ReactHTML.h1
import react.dom.html.ReactHTML.li
import react.dom.html.ReactHTML.ul

private val scope = MainScope()

val App = FC<Props> {
    var shoppingList by useState(emptyList<ShoppingListItem>())

    useEffectOnce {
        scope.launch {
            shoppingList = getShoppingList()
        }
    }

    h1 {
        +"Full-Stack Shopping List"
    }
    ul {
        shoppingList.sortedByDescending(ShoppingListItem::priority).forEach { item ->
            li {
                key = item.toString()
                +"[${item.priority}] ${item.desc} "
            }
        }
    }
}

```

In the snippet above, we use a Kotlin DSL to define the HTML representation of our application, and use `launch` to obtain our list of `ShoppingListItem`s from the API when the component is first initialized.

By making use of React hooks – `useEffectOnce` and `useState`, we can use React's functionality in a concise manner. For more information on how React hooks work, please check out the [official React documentation](https://reactjs.org/docs/hooks-overview.html). To learn more about React with Kotlin/JS, check out the corresponding [hands-on](https://play.kotlinlang.org/hands-on/Building%20Web%20Applications%20with%20React%20and%20Kotlin%20JS/01_Introduction).

If everything has gone according to plan, we can start our application via the Gradle `run` task, and navigate to `http://localhost:9090/` to see our list, nicely formatted:

![image-20200408185054084](./assets/image-20200408185054084.png)

Next, let's allow our users to add new entries to the shopping list, via a text input field.

#### Adding an input field component

In order to receive input from the user, we need an input component which provides us with a callback for when the user submits their entry to the shopping list.

Create the file `src/jsMain/kotlin/InputComponent.kt`, and fill it with the following definition:

```kotlin
import org.w3c.dom.HTMLFormElement
import react.*
import org.w3c.dom.HTMLInputElement
import react.dom.events.ChangeEventHandler
import react.dom.events.FormEventHandler
import react.dom.html.InputType
import react.dom.html.ReactHTML.form
import react.dom.html.ReactHTML.input

external interface InputProps : Props {
    var onSubmit: (String) -> Unit
}

val InputComponent = FC<InputProps> { props ->
    val (text, setText) = useState("")

    val submitHandler: FormEventHandler<HTMLFormElement> = {
        it.preventDefault()
        setText("")
        props.onSubmit(text)
    }

    val changeHandler: ChangeEventHandler<HTMLInputElement> = {
        setText(it.target.value)
    }

    form {
        onSubmit = submitHandler
        input {
            type = InputType.text
            onChange = changeHandler
            value = text
        }
    }
}
```

While an in-depth explanation of how this component works is outside the scope of this hands-on, it can be summarized as follows: The `InputComponent` keeps track of its internal state (what the user has typed so far), and exposes an `onSubmit` handler that gets called when the user submits the form (usually by pressing the `Enter` key).

To use this `InputComponent` from our application. we add the following snippet to `src/jsMain/kotlin/App.kt`,  at the bottom of the `FC` block (after the closing brace for the `ul` element):

```kotlin
InputComponent {
    onSubmit = { input ->
        val cartItem = ShoppingListItem(input.replace("!", ""), input.count { it == '!' })
        scope.launch {
            addShoppingListItem(cartItem)
            shoppingList = getShoppingList()
        }
    }
}
```

When the user submits their text, we create a new `ShoppingListItem`. Its priority is set to be the number of exclamation points in the input, and its description is the input with all exclamation points removed. This turns `Peaches!! 🍑` into a `ShoppingListItem(desc="Peaches 🍑", priority=2)`.

The generated `ShoppingListItem` gets sent to the server via the client we built before. Lastly, we update the UI by obtaining the new list of `ShoppingListItem`s from the server, updating our application state, and letting React do its re-rendering magic.

#### Crossing off items

We currently don't have a way to remove any items from the list again.
Sooner or later, that would result in some *long* lists.
Let's add the ability to remove items from our list.

Rather than add another UI element (like a "delete" button), we can just modify our existing list: when the user clicks one of the items in the list, we delete it. To achieve this, we can just pass a corresponding handler to the `attrs.onClickFunction` of our list elements.

In `src/jsMain/kotlin/App.kt`, add the following to the `li` block (inside the `ul` block):

```kotlin
onClick = {
    scope.launch {
        deleteShoppingListItem(item)
        shoppingList = getShoppingList()
    }
}
```

We once again invoke our API client with the element that should be removed, and ask the server for an updated shopping list, which re-renders our user interface.

Now seems like a good time to give our application another spin. We can start it using the Gradle `run` task once more, navigate to `http://localhost:9090/`, and can try adding and removing elements in the list.

![](/assets/finished.gif)

At this point, the application is pretty much done! The only current problem is that we can't _persist_ our data, meaning that our shopping list is going to vanish the moment we terminate the server process.

In the next chapter, we'll explore how to use MongoDB as an easy way to store and retrieve our shopping list items even after the server gets shut down.

### Relevant Gradle configuration

To provide all the functionality used in this section, we need to include a number of libraries from the Kotlin and JavaScript (npm) ecosystems. Please refer to the `jsMain` dependency block in the `build.gradle.kts` file to see the full setup.
