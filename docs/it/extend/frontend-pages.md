# Frontend Pages and Resolvers

As explained in the [Routes and Content](routes.md#frontend-routes) documentation, we can use Mithril's routing system to show different [components](frontend.md#components) for different routes. Mithril allows you to use any component you like, even a Modal or Alert, but we recommend sticking to component classes that inherit the `Page` component.

## The Page Component

We provide `flarum/components/Page` as a base class for pages in both the `admin` and `forum` frontends. It has a few benefits:

- Automatically updates [`app.current` and `app.previous` PageState](#pagestate) when switching from one route to another.
- Automatically closes the modal and drawer when switching from one route to another.
- Applies `this.bodyClass` (if defined) to the '#app' HTML element when the page renders.
- It's also good for consistency's sake to use a common base class for all pages.
- If the page's `scrollTopOnCreate` attribute is set to `false` in `oninit`, the page won't be scrolled to the top when changed.
- If the page's `useBrowserScrollRestoration` is set to `false` in `oninit`, the browser's automatic scroll restoration won't be used on that page.

Page components work just like any other inherited component. For a (very simple) example:

```js
import Page from 'flarum/components/Page';


export default class CustomPage extends Page {
  view() {
    return <p>Hello!</p>
  }
}
```

## PageState

Sometimes, we want to get information about the page we're currently on, or the page we've just come from.
To allow this, Flarum creates (and stores) instances of [`PageState`](https://api.docs.flarum.org/js/master/class/src/common/states/pagestate.js~pagestate) as `app.current` and `app.previous`.
These store:

- The component class being used for the page
- A collection of data that each page sets about itself. The current route name is always included.

Data can be set to, and retrieved from, Page State using:

```js
app.current.set(KEY, DATA);
app.current.get(KEY);
```

For example, this is how the Discussion Page makes its [`PostStreamState`](https://api.docs.flarum.org/js/master/class/src/forum/states/poststreamstate.js~poststreamstate) instance globally available.

You can also check the type and data of a page using `PostStreamState`'s `matches` method. For instance, if we want to know if we are currently on a discussion page:

```jsx
import IndexPage from 'flarum/components/DiscussionPage';
import DiscussionPage from 'flarum/components/DiscussionPage';

// To just check page type
app.current.matches(DiscussionPage);

// To check page type and some data
app.current.matches(IndexPage, {routeName: 'following'});
```

## Route Resolvers (Advanced)

[Advanced use cases](https://mithril.js.org/route.html#advanced-component-resolution) can take advantage of Mithril's [route resolver system](https://mithril.js.org/route.html#routeresolver).
Flarum actually already wraps all its components in the `flarum/resolvers/DefaultResolver` resolver. This has the following benefits:

- It passes a `routeName` attr to the current page, which then provides it to `PageState`
- It assigns a [key](https://mithril.js.org/keys.html#single-child-keyed-fragments) to the top level page component. When the route changes, if the top level component's key has changed, it will be completely rerendered (by default, Mithril does not rerender components when switching from one page to another if both are handled by the same component).

### Using Route Resolvers

There are actually 3 ways to set the component / route resolver when registering a route:

- the `resolver` key can be used to provide an **instance** of a route resolver. This instance should define which component should be used, and hardcode the route name to be passed into it. This instance will be used without any modifications by Flarum.
- The `resolverClass` key AND `component` key can be used to provide a **class** that will be used to instantiate a route resolver, to be used instead of Flarum's default one, as well as the component to use. Its constructor should take 2 arguments: `(component, routeName)`.
- The `component` key can be used alone to provide a component. This will result in the default behavior.

For example:

```js
// See above for a custom page example
import CustomPage from './components/CustomPage';
// See below for a custom resolver example
import CustomPageResolver from './resolvers/CustomPageResolver';

// Use a route resolver instance
app.routes['resolverInstance'] = {path: '/custom/path/1', resolver: {
  onmatch: function(args) {
    if (!app.session.user) return m.route.SKIP;

    return CustomPage;
  }
}};

// Use a custom route resolver class
app.routes['resolverClass'] = {path: '/custom/path/2', resolverClass: CustomPageResolver, component: CustomPage};

// Use the default resolver class (`flarum/resolvers/DefaultResolver`)
app.routes['resolverClass'] = {path: '/custom/path/2', component: CustomPage};
```

### Custom Resolvers

We strongly encourage custom route resolvers to extend `flarum/resolvers/DefaultResolver`.
For example, Flarum's `flarum/resolvers/DiscussionPageResolver` assigns the same key to all links to the same discussion (regardless of the current post), and triggers scrolling when using `m.route.set` to go from one post to another on the same discussion page:

```js
import DefaultResolver from '../../common/resolvers/DefaultResolver';

/**
 * This isn't exported as it is a temporary measure.
 * A more robust system will be implemented alongside UTF-8 support in beta 15.
 */
function getDiscussionIdFromSlug(slug: string | undefined) {
  if (!slug) return;
  return slug.split('-')[0];
}

/**
 * A custom route resolver for DiscussionPage that generates the same key to all posts
 * on the same discussion. It triggers a scroll when going from one post to another
 * in the same discussion.
 */
export default class DiscussionPageResolver extends DefaultResolver {
  static scrollToPostNumber: number | null = null;

  makeKey() {
    const params = { ...m.route.param() };
    if ('near' in params) {
      delete params.near;
    }
    params.id = getDiscussionIdFromSlug(params.id);
    return this.routeName.replace('.near', '') + JSON.stringify(params);
  }

  onmatch(args, requestedPath, route) {
    if (route.includes('/d/:id') && getDiscussionIdFromSlug(args.id) === getDiscussionIdFromSlug(m.route.param('id'))) {
      DiscussionPageResolver.scrollToPostNumber = parseInt(args.near);
    }

    return super.onmatch(args, requestedPath, route);
  }

  render(vnode) {
    if (DiscussionPageResolver.scrollToPostNumber !== null) {
      const number = DiscussionPageResolver.scrollToPostNumber;
      // Scroll after a timeout to avoid clashes with the render.
      setTimeout(() => app.current.get('stream').goToNumber(number));
      DiscussionPageResolver.scrollToPostNumber = null;
    }

    return super.render(vnode);
  }
}
```