# Komposition

<p class="description">Die Material-UI versucht die Komposition so einfach wie möglich zu gestalten.</p>

## Wrapping components

In order to provide the maximum flexibility and performance, we need a way to know the nature of the child elements a component receives. To solve this problem we tag some of our components when needed with a `muiName` static property.

You may, however, need to wrap a component in order to enhance it, which can conflict with the `muiName` solution. If you wrap a component verify if that component has this static property set.

If you encounter this issue, you need to use the same tag for your wrapping component that is used with the wrapped component. In addition, you should forward the properties, as the parent component may need to control the wrapped components props.

Let's see an example:

```jsx
const WrappedIcon = props => <Icon {...props} />;
WrappedIcon.muiName = Icon.muiName;
```

{{"demo": "pages/guides/composition/Composition.js"}}

## Component property

Material-UI allows you to change the root node that will be rendered via a property called `component`.

### How does it work?

The component will render like this:

```js
return React.createElement(this.props.component, props)
```

For example, by default a `List` component will render a `<ul>` element. This can be changed by passing a [React component](https://reactjs.org/docs/components-and-props.html#function-and-class-components) to the `component` property. The following example will render the `List` component with a `<nav>` element as root node instead:

```jsx
<List component="nav">
  <ListItem>
    <ListItemText primary="Trash" />
  </ListItem>
  <ListItem>
    <ListItemText primary="Spam" />
  </ListItem>
</List>
```

This pattern is very powerful and allows for great flexibility, as well as a way to interoperate with other libraries, such as [`react-router`](#react-router-demo) or your favorite forms library. But it also **comes with a small caveat!**

### Caveat with inlining

Using an inline function as an argument for the `component` property may result in **unexpected unmounting**, since you pass a new component to the `component` property every time React renders. For instance, if you want to create a custom `ListItem` that acts as a link, you could do the following:

```jsx
import { Link } from 'react-router-dom';

const ListItemLink = ({ icon, primary, secondary, to }) => (
  <li>
    <ListItem button component={props => <Link to={to} {...props} />}>
      {icon && <ListItemIcon>{icon}</ListItemIcon>}
      <ListItemText inset primary={primary} secondary={secondary} />
    </ListItem>
  </li>
);
```

⚠️ However, since we are using an inline function to change the rendered component, React will unmount the link every time `ListItemLink` is rendered. Not only will React update the DOM unnecessarily, the ripple effect of the `ListItem` will also not work correctly.

The solution is simple: **avoid inline functions and pass a static component to the `component` property** instead. Let's change our `ListItemLink` to the following:

```jsx
import { Link } from 'react-router-dom';

class ListItemLink extends React.Component {
  renderLink = React.forwardRef((itemProps, ref) => (
    // mit react-router-dom@^5.0.0 benutze `ref` anstatt `innerRef`
    <RouterLink to={this.props.to} {...itemProps} innerRef={ref} />
  ));

  render() {
    const { icon, primary, secondary, to } = this.props;
    return (
      <li>
        <ListItem button component={this.renderLink}>
          {icon && <ListItemIcon>{icon}</ListItemIcon>}
          <ListItemText inset primary={primary} secondary={secondary} />
        </ListItem>
      </li>
    );
  }
}
```

`renderLink` will now always reference the same component.

### Caveat with shorthand

You can take advantage of the properties forwarding to simplify the code. In this example, we don't create any intermediary component:

```jsx
import { Link } from 'react-router-dom';

<ListItem button component={Link} to="/">
```

⚠️ However, this strategy suffers from a little limitation: properties collision. The component providing the `component` property (e.g. ListItem) might not forward all its properties to the root element (e.g. dense).

### React Router Demo

Here is a demo with [React Router DOM](https://github.com/ReactTraining/react-router):

{{"demo": "pages/guides/composition/ComponentProperty.js"}}

### With TypeScript

You can find the details in the [TypeScript guide](/guides/typescript#usage-of-component-property).

### Vorbehalt bei Refs

Einige Komponenten wie `ButtonBase` (und daher `Button`) benötigen Zugriff auf den darunterliegenden DOM-Knoten. Dies wurde zuvor mit `ReactDOM.findDOMNode(this)` durchgeführt. `FindDOMNode` ist jedoch veraltet (wodurch es im React Concurrent Modus nicht anwendbar ist), zugunsten der Komponenten Refs und ref - Forwarding.

Es ist daher erforderlich, dass die Komponente, die Sie an die `Komponente` Eigenschaft übergeben, einen Ref halten kann. Dazu gehören:

- Klassen-Komponenten
- ref Weiterleitungskomponenten (`React.forwardRef`)
- eingebaute Komponenten zB `div` oder `a`

Ist dies nicht der Fall, geben wir eine Warnung ähnlich der folgenden aus:

> Invalid prop `component` supplied to `ComponentName`. Es wurde ein Elementtyp erwartet, der eine Referenz enthalten kann.

Zusätzlich gibt React eine Warnung aus.

Sie können diese Warnung beheben, indem Sie `React.forwardRef` verwenden. Weitere Informationen dazu finden Sie in [dieser Sektion in den offiziellen React-Dokumenten](https://reactjs.org/docs/forwarding-refs.html).

Um herauszufinden, ob die Material-UI - Komponente, die Sie verwenden, diese Anforderung hat, überprüfen Sie API - Dokumentation für diese Komponente. Wenn Sie Refs weiterleiten müssen, wird die Beschreibung mit diesem Abschnitt verknüpft.

### Vorsicht bei StrictMode oder unstable_ConcurrentMode

Wenn Sie Klassenkomponenten an die `Komponente` Eigenschaft übergeben und nicht im strikten Modus laufen, Sie müssen nichts ändern, da wir `ReactDOM.findDOMNode` sicher verwenden können. Bei Funktionskomponenten müssen Sie jedoch Ihre Komponente in `React.forwardRef` einhüllen:

```diff
- const MyButton = props => <div {...props} />
+ const MyButton = React.forwardRef((props, ref) => <div {...props} ref={ref} />)
<Button component={MyButton} />
```