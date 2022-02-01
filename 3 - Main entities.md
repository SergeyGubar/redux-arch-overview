## Most important entities

**ReduxAction** - an interface for all the actions in the system, which can be handled by the redux store. 

**AppState** - application state, which describes all the data in an app. Implemented as a data class with references to other state classes 
for a better decomposing.

**Reduce** - singleton object which is used as a root for all reducers (added using extension property and delegation to **Reducer** class).

**Reduce.appState** - root extension property for combining all the reducers into a single place.

**Store** - state container, which holds a reference to the store and is responsible for handling actions, creating a new state using the app reducer,
and notifying all the middlewares about new action.

**Core** - container for a store, which holds references to all the connectors and is responsible for wiring up the specified screen with the store 
(connecting the connector, so it starts mapping props). Also, core holds a weak reference to the current activity, which can be used for all types of 
side effects (dialogs, snackbars, routing, etc).

**Configurator** - base class for entities, responsible for subscribing and unsubscribing UI components. Stores related connectors in a fields,
and gets notified about every UI component that needs ```Props```  

**BaseConnector** - base class for all connectors. Defines only one method ```map``` which is used to describe how to create a data which will be later
displayed on the UI. 

**BaseFragment** - base class for UI component of the application. Holds a lot of properties with the most important - ```props``` property.
Props are implemented as a ```LiveData```, so they are lifecycle aware, which means that the screen won't be re-rendered if it's not currently visible to the user.
Also, there are several different implementations like ```BaseDialogFragment``` for different types of UI element, but all of them have the similar structure.

**Middleware** - base class for the entity, which is responsible for triggering side effects based on the actions flow. Can hold reference to the ```Store``` or ```Core```
and dispatches actions based on side effect logic flow (for example, make a request to the server, and dispatch result to the store).

## Data flow diagram

![redux](https://user-images.githubusercontent.com/15278596/110431174-1c2c4a80-80b6-11eb-86ab-fe01fa0b16cf.png)

