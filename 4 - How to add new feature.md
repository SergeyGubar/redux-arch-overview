## 4. New feature implementation process (example)

New feature implementation process in Redux is standardized, and almost every feature requires the same typical steps.
Let's take the following example - you are developing screen with the information about user (name, phone number, password, etc) with ability to change it and 
sync to the server.
So what do you need to implement this feature?

### 4.1 Designing props

First of all, let's design our props:

```kotlin
data class LoginScreenProps(
    val firstName: String?,
    val lastName: String?,
    val email: String?,
    val phone: String?,
    val isFirstNameValid: Boolean,
    val isLastNameValid: Boolean,
    val isPhoneValid: Boolean,
    val firstNameChanged: Command.With<String>,
    val lastNameChanged: Command.With<String>,
    val phoneChanged: Command.With<String>,
    val save: Command?,
)
```

This data class contains all the information, which is needed to render our screen - ```firstName```, ```lastName```, ```email``` and ```phone```
are used to render our input fields. ```isFirstNameValid, isLastNameValid``` - computed properties to show some validation (e.g if fields are empty).

Also, we need to wire our input field with the redux store to add interactions - we use ```Command.With<String>``` for it. Basically, this type notes 
that it's a closure, which takes string as an input. We pass the string from the text fields values in a listener, so our store knows about any changes of these fields.

And lastly, we add the ```save``` command, which is a closure, which will invoke when the user clicks save button. This command is optional, and this semantically
means that this command can be empty - in case if user didn't enter any data, we don't allow to save it by passing null instead of command.

### 4.2 Designing state and connector

The next step is to implement our ```State``` class and ```Connector```.
For the state we can also use simple data class:

```kotlin
data class LoginState(
    val isDirty: Boolean = false,
    val firstName: String? = null,
    val lastName: String? = null,
    val email: String? = null,
    val phone: String? = null,
    val isFirstNameValid: Boolean = true,
    val isLastNameValid: Boolean = true,
    val isPhoneValid: Boolean = true
)
```

Every state should be nested as a field into the ```AppState``` - so we need to add it as a property.
Also, every state should have default params which denote the empty state.

```Connector``` is the entity, which maps our state in our props. To create it - just extend the ```BaseConnector``` class and implement the ```map``` function.

```kotlin
class LoginConnector(private val core: Core) : BaseConnector<LoginProps>() {

    override fun map(appState: AppState): LoginProps {
        with(appState.aboutYouState) {
            return LoginProps(
                firstName = firstName,
                lastName = lastName,
                email = email,
                phone = phone,
                isFirstNameValid = isFirstNameValid,
                isLastNameValid = isLastNameValid,
                isPhoneValid = isPhoneValid,
                firstNameChanged = Command.With.nop(),
                lastNameChanged = Command.With.nop(),
                phoneChanged = Command.With.nop(),
                save = Command.nop()
            )
        }
    }
}
```

After writing these entities, we should be able to wire all of these together and see how it works.
Now we need to register our connector in one of the existing screen configurators, or add a new configurator
(if it is a completely new functionality in app).
Configurators are stored in a ```Core``` class, and are notified about every UI component, that needs to be configured.
In a configurator class:

```kotlin
fun subscribe(fragment: Fragment) {
        val hashCode = fragment.hashCode()
        when (fragment) {
            is AboutYouFragment -> connect(
                aboutYouConnector,
                { fragment.props.value = it },
                hashCode = hashCode
            )
        }
}
```

Don't forget to disconnect your screen, when it becomes not visible to user:
```kotlin
fun unsubscribe(fragment: Fragment) {
        val hashCode = fragment.hashCode()
        when (fragment) {
            is AboutYouFragment -> disconnect(
                aboutYouConnector,
                hashCode = hashCode
            )
        }
}
```

At this point, when you open the fragment, it should get correct props which are mapped from the state. To test it - write some default values in state class,
and check whether fragment's  ```render``` method gets the right value.

### 4.3 Implementing the interactions

After we've wired our screen with the state, let's add user interactions to it.
Let's create the following actions:

```kotlin
data class AccountFirstNameChanged(val newValue: String) : ReduxAction
data class AccountLastNameChanged(val newValue: String) : ReduxAction
data class AccountPhoneChanged(val newValue: String) : ReduxAction
object UpdateAccountInfo: ReduxAction
```

These actions stand for first name input change, second name change and phone number input change respectively. All of them accept new value as a string,
so we can propagate it to our state. Also, we've added ```UpdateAccountInfo``` action to notify the app when user wants to save these changes.

Now let's implement the reducer - we do it by adding an extension property to ```Reduce``` object and delegating the implementation to ```Reducer``` class:

```kotlin
val Reduce.loginState by Reducer<LoginProps> { state, action ->
    when (action) {
        is AccountFirstNameChanged ->
            state.copy(
                firstName = action.newValue,
                isFirstNameValid = Validator.isValidFirstName(action.newValue)
            )
        is AccountLastNameChanged ->
            state.copy(
                lastName = action.newValue,
                isLastNameValid = Validator.isValidLastName(action.newValue)
            )
        is AccountPhoneChanged ->
            state.copy(
                phone = action.newValue,
                isPhoneValid = Validator.isValidPhone(action.newValue)
            )
        SignedOut -> AboutYouState()
        else -> state
    }
}

``` 

Few moments to mention - first of all, we need to handle all actions that will be dispatch later, and implement the related logic. Reducer can't change 
the immutable state, so we need to return new object by using ```copy``` and changing the needed fields.

Also, we can handle not only 'about you' actions in our reducer, but all app actions - each reducer gets notified about any action which is triggered in the application.
So, we can handle ```SignedOut``` for example, and return empty state to clear our data if user logged out.


After implementing the reducer, we need to add it in ```AppReducer``` so it starts changing our state when actions occur.
Now, getting back to our connector, we can replace the hardcoded commands with the following code:

```kotlin
    ...
    firstNameChanged = core.store.dispatchWith { AccountFirstNameChanged(it) },
    lastNameChanged = core.store.dispatchWith { AccountLastNameChanged(it) },
    phoneChanged = core.store.dispatchWith { AccountPhoneChanged(it) },
    save = if (isDirty && isFirstNameValid && isLastNameValid &&
        (isPhoneValid && phone.getDigits().length > PHONE_NUMBER_LENGTH)
    ) core.store.bind(UpdateAccountInfo)
    else null,
    ...
```

```bind``` and ```dispatchWith``` are just convenience function which create a ```Command.With<T>``` and ```Command``` respectively, which dispatches
the related action to store.

After this step, we've got everything up and running, except for the last part - making a request to the backend to save the data.

## 4.4 Side effects handling

In our case, we want to make a network request, which saves the user data to the backend.
One possible solution - make it inside closure within the connector. However, it makes our connectors pretty big and doesn't allow to share the logic
between different parts of the application.

To implement this kind of side effect - we will use ```Middleware```.

Creating ```Middleware``` is simple - just extend the base interface and implement the ```apply``` method.
After extending the interface, we need to check if this middleware is interested in the upcoming action, and perform some side effects, if needed.
Also, every ```apply``` can't perform any work in it, because it's called on the main thread (that's why we use coroutines in this example) and it can't return anything - so the only way to
communicate with our app is to dispatch an action (```AccountInfoUpdated``` in this example to the store).

```kotlin
class LoginMiddleware(private val store: Store, private val loginService: LoginService): Middleware {
    override fun apply(action: ReduxAction) {
        when (action) {
            is UpdateAccountInfo -> {
                launch {
                    val loginState = store.appState().loginState
                    aboutYouService.updateInfo(aboutYouState.phone, aboutYouState.name, aboutYouState.lastName, onSuccess = {
                        store.dispatch(AccountInfoUpdated)
                    })
                }
            }   
        }       
    }
}
```

After implementing the middleware, register it in the ```Store``` class, and the feature is implemented.
