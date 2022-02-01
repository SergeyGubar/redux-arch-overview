## Redux basics

Redux is an architecture with unidirectional data flow, which is focused on a **state management**.

Basically, the whole concept of Redux is to extract an application state into a **State** object, which can be modified only with special entities (called **Reducers**).
Reducers can only be called by dispatching an **Action** into the Redux store, and each action dispatch with subsequent state reduce creates a new **Props** for each UI component to render.