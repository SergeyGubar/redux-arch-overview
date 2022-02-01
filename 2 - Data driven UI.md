## 2. Data driven UI

The concept of data driven UI is pretty straightforward - the whole state of a view should be described in a structure (conventionally called **Props**),
and the only way of changing the UI is to create a new instance of **Props**. 

The main advantage of data driven UI is that it's deterministic and predictable - you can always understand which type and content of props is
currently displayed on the UI. Also, this concepts simplifies UI debugging because the flow of the events which led to the particular UI state 
is not important - all you need to know is the latest value of the props.  