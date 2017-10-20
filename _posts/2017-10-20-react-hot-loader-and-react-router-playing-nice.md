---
layout: post
title: "React Hot Loader and React Router playing nice"
description: "Issues that I've encountered while setting up React Hot Loader and React Router caused by an attempt to uprade to React 16"
date: 2017-10-20
tags: [react, javascript, typescript]
comments: false
share: true
---

Recently I've been reading about all the sweet improvements in [React 16](https://reactjs.org/blog/2017/09/26/react-v16.0.html). Naturally I got really excited and wanted to upgrade one of the projects that I've been working on. Turns out the version of React Hot Loader that I was using (`1.x`) wasn't really compatible with React 16, so I had to upgrade it as well. I went ahead and switched to the latest version of React Hot Loader (`3.1.1` at the time of writing this) and for anyone that tried to do the same, you know that the API in version `3.x` had changed a bit (so I had to re-write a bunch of stuff). I decided to write this down in the hopes that someone might find it useful.

No problem, the documentation for React Hot Loader had very nice code examples and I re-wrote my app setup in no time.

This was my first attempt (note that the project is written in `typescript`):

```typescript
// Boot.tsx (entry point for the app)

const createStoreWithMiddleware = applyMiddleware(
    thunkMiddleware,
    createLogger
)(createStore);
const store = createStoreWithMiddleware(rootReducer);

const container: HTMLElement = document.getElementById("app-container");

ReactDOM.render(
    <AppContainer>
        <Root store={store} history={browserHistory} />
    </AppContainer>,
    container
);

if (module.hot) {
    module.hot.accept("./Root", () => {
        const NextRoot = require<{ default: typeof Root }>("./Root").default;
        ReactDOM.render(
            <AppContainer>
                <NextRoot store={store} history={browserHistory} />
            </AppContainer>,
            container
        );
    });
}
```

However, I got the following warning (this is React Router `3.3` by the way):

`Warning: [react-router] You cannot change <Router routes>; it will be ignored`

This is how my `Root` component looked like:

```typescript
// Root.tsx

// not the actual routes - but you get the idea
const routes: JSX.Element = (
    <Route component={App}>
        <Redirect from="/" to="sample" />
        <Route component={SampleComponent} path="/sample" />
        <Route component={SomeOtherSampleComponent} path="/sample/:category" />
    </Route>
);

interface RootProps {
    store: Store<any>;
    history: HistoryBase;
}

const Root: React.SFC<RootProps> = (props: RootProps) => (
    <Provider store={props.store}>
        <Router history={props.history}>
            {routes}
        </Router>
    </Provider>
);
export default Root;
```

I tried a bunch of things to get rid of the warning and what worked for me at the end is changing the `Root` component to look like:

```typescript
// Root.tsx

interface RootProps {
    store: Store<any>;
    history: HistoryBase;
}

export default class Root extends React.Component<RootProps, any> {
    private routes: JSX.Element = (
        <Route component={App}>
            <Redirect from="/" to="sample" />
            <Route component={SampleComponent} path="/sample" />
            <Route component={SomeOtherSampleComponent} path="/sample/:category" />
        </Route>
    );

    render() {
        return (
            <Provider store={this.props.store}>
                <Router history={this.props.history}>
                    {this.routes}
                </Router>
            </Provider>
        );
    }
}
```

Voila! Everything worked fine afterwards! Yay!

The upgrade to React 16 was pretty smooth experience overall, and it was totally worth it - the app file size reduced considerably and the performance improvements I'm seeing are in the range of 20-30%. Pretty awesome, really.