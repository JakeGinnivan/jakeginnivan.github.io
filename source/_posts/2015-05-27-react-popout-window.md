---
layout: post
published: true
title: ReactJS Popout window
comments: true
---

Recently I had a requirement to have a chat history panel which could be popped out into it's own window and that window needed to have live updates. I figured I had two ways of doing it, I could open a new window which initialised itself and had it's own connection to the server to get updates or I could keep with the React way and have the components which pop out simply flow updates to the window via props.

The component which needs to support popping out is the `History` component which takes a set of items which get rendered.
``` js
<ChatHistory items={this.props.request.history} />
```

A simplified implementation looks like this

``` js
class ChatHistory extends React.Component {
  constructor(props) {
    super(props)
    this.popout = this.popout.bind(this)
    this.state = {
      poppedOut: false
    }
  }

  popout() {
    this.setState({ poppedOut: true });
  }

  render() {
    return (
      <div>
        <Graphicon style={{float:right}} icon='popout' onClick={this.popout} />
        {this.props.items.map(item =>
          <div>
            {item.date}
            <pre>{item.text}</pre>
          </div>
        )}
      </div>
    )
  }
}
```

Next is how do I pop this content out. First I create a `PoppedOutChatHistory` component:

``` js
class PoppedOutChatHistory extends React.Component {
  constructor(props) { super(props) }

  render() {
    return (
      <div>
        {this.props.messages.map(item =>
          <div>
            {item.date}
            <pre>{item.text}</pre>
          </div>
        )}
      </div>
    )
  }
}

```

Then I introduced a `Popout` react component into the `ChatHistory` render function:

``` js
render() {
  if (this.state.isPoppedOut) {
    return (
      <Popout title='Window title' onClosing={this.popupClosed}>
        <PoppedOutChatHistory messages={this.props.items} />
      </Popout>
    );
  } else {
    return (
      <div>
      <span style={{float:right}} onClick={this.popout} className="buttonGlyphicon glyphicon glyphicon-export"></span>
        {this.props.messages.map(item =>
          <div>
            {item.date}
            <pre>{item.text}</pre>
          </div>
        )}
      </div>
    );
  }
}

popupClosed() {
  this.setState({ poppedOut: false });
}
```

The `Popout` component is quite simple, when `show` is set to true then it will render the content in a new popout window. The nice thing about having this as a React component is that the `Popout` window's lifecycle is tied to the lifecycle of the owning component. So when the History component is unmounted the `Popout` will close the popup it might have had opened.

## Summary
I am pretty happy with the result of this Component, I can continue to flow props and handle events. For instance if I wanted to pop up a input form:

``` js
render() {
  return (
    <Popout title='Window title' onClosing={this.popupClosed}>
      <input type='text' onChange={this.textChanged} />
      <input type='button' onClick={this.submit} />
    </Popout>
  );
}
```

### Source and install

 > npm install react-popup --save

[Source on GitHub](https://github.com/JakeGinnivan/react-popout)
