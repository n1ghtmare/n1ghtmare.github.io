---
layout: post
title: "Implement a keyboard shortcuts handler in typescript"
description: "In this post we'll look at a way to write your own keyboard shortcuts handler (similar to hotkeys.js), that will capture user input and execute callbacks, as needed"
date: 2022-01-14
tags: [programming, typescript, javascript, tutorial]
comments: false
share: true
---

A new feature I wanted to add to a [next.js](https://github.com/n1ghtmare/dotfiles) project that I'm working on, is the support for keyboard shortcuts for various actions within the application (navigation, commands, focus on UI elements etc.). I needed support for key sequences, key combinations, as well as a mixture of both. So `a b`, `c+b`,`a b a+b` as well as `a+b a+b` are all valid and should work. In addition, I needed shortcut scopes. For example you could have different hotkeys defined for different parts of your UI and enable/disable them in accordance to your state. When I open a modal, for example, I would like it to work within its own hotkey scope and prevent triggering hotkeys for other parts of the app.

Before I wrote my own implementation, I looked around to see what projects are available, unfortunately I found either abandoned (read: unmaintained) libraries, or libraries that don't cover my requirements (for example the popular [hotkeys.js](https://github.com/jaywcjlove/hotkeys) doesn't support key sequences, at the time of writing this). Also, I thought it would be a fun exercise to write my own tiny library to handle things.

So, I started with laying out how my api should look like:

```typescript
const { unbind } = registerHotkey("a b a+b", "hotkey-scope", (e: KeyboardEvent) => {
    // prevent the default if needed, so you don't collide with the browser shortcuts
    e.preventDefault(); 

    console.log("triggered!");
})
```

The `unbind` function would be called upon cleanup, to unregister/clear the hotkey. In a react app that would be something like:

```typescript

useEffect(() => {
    const { unbind } = registerHotkey("a b", "global", (e: KeyboardEvent) => {
        console.log("it's working")
    });

    return () => {
        unbind();
    }
}, []);
```
First let's look at some of the types that I've defined. I decided to use a (sort of) [Trie](https://en.wikipedia.org/wiki/Trie) data structure to store the keys with their callbacks. Where each leaf node would contain a callback that must be called once reached (through traversal for each key press), this will make sense a little later. A node would look like this:

```typescript
type HotkeyNode = {
    children: Map<string, HotkeyNode>;
    callback?: (e: KeyboardEvent) => void;
};
```

Another data structure would be the hotkey scope:

```typescript
type HotkeyScope = {
    root: HotkeyNode;
    currentNode: HotkeyNode;
};
```
Here we keep track of the entire tree (under root) as well as keep track of the current node that we're on while traversing and looking for key matches.

First we create a scope with a particular name, we also need to keep track of the current scope that is being used, I decided to have one called `global` as a default:

```typescript
let currentScopeName = "global";
const scopeMap = new Map<string, HotkeyScope>();

function createScopeIfNeeded(scopeName: string) {
    const scope = scopeMap.get(scopeName);

    if (scope) {
        return scope;
    }

    const root: HotkeyNode = {
        children: new Map<string, HotkeyNode>(),
        callback: null
    };

    const newScope: HotkeyScope = { root, currentNode: root };
    scopeMap.set(scopeName, newScope);

    return newScope;
}
```
If there is no scope defined we create one to hold our (still empty) tree structure. So far so good.

Another thing we'll need is to process the input of the user when registering a hotkey, here is how this might look like:

```typescript
const hotkeys = "a b c+a";
const processedHotkeys: string[] = hotkeys.split(" ").map((x) =>
    x
        .split("+")
        .map((y) => normalizeKey(y))
        .sort()
        .join("+")
);
// processedHotkeys -> ["a", "b", "a+c"]
```
You might notice 2 things here, `normalizeKey(key)` and the `sort()` that we do. That is so that when a user presses a combination of keys we're not dependant on the order in which he pressed them (`a+b` should be the same as `b+a`). The normalize function is there to ensure that we handle some edge cases that are dependant on OSs and different browser engines (for example `Ctrl` is sometimes `Control` and sometimes `Ctrl` dependant on the platform you're on), for now, I keep only a few keys there, but this function could be expanded to cover more in the future (not sure of all the edge cases yet), this covers pretty much all of my needs right now:

```typescript
function normalizeKey(hotkey: string) {
    // make all keys lower before
    hotkey = hotkey.trim().toLowerCase();
    switch (hotkey) {
        case "esc":
            return "escape";
        case "ctrl":
            return "control";
        case "option":
            return "alt";
        default:
            return hotkey;
    }
}
```

Here is how we would do an insert into our tree data structure:

```typescript
function insertIntoTree(
    scope: HotkeyScope,
    hotkeys: string[],
    callback: (e: KeyboardEvent) => void
) {
    let workingNode: HotkeyNode = scope.root;

    for (const hotkey of hotkeys) {
        const { children } = workingNode;
        if (!children?.get(hotkey)) {
            // create a node only if it's not already in the tree
            const node: HotkeyNode = {
                children: new Map<string, HotkeyNode>()
            };
            children.set(hotkey, node);
        }
        workingNode = children.get(hotkey);
    }

    // we've reached the end - attach the callback
    workingNode.callback = callback;
}
```

For each of our processed hotkeys we traverse the tree starting from the root and create a new node (if it doesn't exist yet). If we've reached the final hotkey we attach a callback.

Now that we have a way to insert, we also need a way to remove a node from the tree, this is a bit more involved for the data structure that I'm using, but we're going to be doing this only when unbinding the registered hotkey so that should be fine. Here it is:

```typescript
function removeFromTree(scope: HotkeyScope, hotkeys: string[]) {
    let workingNode: HotkeyNode = scope.root;
    const chain: HotkeyNode[] = [];

    // traverse down the trie & build a chain of nodes that need to be checked for removal
    for (const hotkey of hotkeys) {
        const { children } = workingNode;
        if (!children) {
            return;
        }

        const childNode: HotkeyNode = children.get(hotkey);
        if (!childNode) {
            return;
        }

        chain.push(workingNode);
        workingNode = childNode;
    }

    if (chain.length === 0) {
        // shouldn't happen (hotkey not found)
        return;
    }

    let lastNode: HotkeyNode = chain.pop();

    if (lastNode.children.get(hotkeys[hotkeys.length - 1]).children.size > 0) {
        // node continues deeper, we're not interested of removing it
        return;
    }

    for (let i = hotkeys.length - 1; i >= 0; i--) {
        lastNode.children.delete(hotkeys[i]);

        if (lastNode.children.size > 0) {
            // there are more children (we're in a branch, don't continue removing the parent)
            return;
        }

        lastNode = chain.pop();
    }
}
```
Basically, we traverse the tree untill we find the node we're looking for, we store each of the nodes we've traversed into an array (`chain`), then we go back through the chain and see if the node has a sibling, if it does, then we stop there since we don't want to remove nodes that have different path than the one we're currently removing. For example - if we have `a b c` and `a b d` and we want to delete the latter, we delete only the node containing `d` and leave `a` and `b` in tact.

Cool. Next, we will build a way to search for a node (one key at a time, not a full search):

```typescript
function searchCurrentNodeForHotkey(scope: HotkeyScope, hotkey: string): HotkeyNode {
    const { currentNode } = scope;

    if (!currentNode) {
        return null;
    }

    const { children } = currentNode;

    if (!children.get(hotkey)) {
        return null;
    }

    return children.get(hotkey);
}
```

This is looking only at the current node (we're not starting at the root), don't worry it will all make sense if it's a bit confusing at the moment.

Another thing we would need is a way reset the `currentNode` a certain amount of time after the user has stopped typing, so here is a small debounce utility function that would help us execute a callback function after a certain amount of time (milliseconds), you will see how this is used in a bit:

```typescript
function debounce(fn: () => void, milliseconds: number) {
    let timeoutId = null;

    return () => {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(fn, milliseconds);
    };
}
```

Now that we have all that setup, let's get to the fun stuff, here is how hour `keydown` listener would look like:

```typescript
let buffer: string[] = [];

function createKeydownListener(debounceTimeInMilliseconds: number) {
    // Reset the current node back to start traversing from the root 
    // after a the debounce time
    const resetCurrentNodeDebounced = debounce(() => {
        const scope: HotkeyScope = scopeMap.get(currentScopeName);

        if (scope) {
            scope.currentNode = scope.root;
        }
    }, debounceTimeInMilliseconds);

    return (e: KeyboardEvent) => {
        // We detect if the user has pressed the key and keeps holding it down
        if (e.repeat) {
            return;
        }

        const scope: HotkeyScope = scopeMap.get(currentScopeName);

        // ensure that a scope is defined
        if (!scope) {
            return;
        }

        resetCurrentNodeDebounced();

        // push the pressed key to a buffer (after we normalize it)
        // we remove entries from the buffer on keyup
        // so the buffer is used only for key combinations
        buffer.push(normalizeKey(e.key));

        // search if we can find a match
        const node: HotkeyNode = searchCurrentNodeForHotkey(scope, buffer.sort().join("+"));

        if (node === null) {
            // nothing was found, the key was not registered 
            return;
        }

        if (node.callback) {
            // if there is a callback it means we're at a leaf node
            // execute the callback
            node.callback(e);

            // reset we found what we're looking for
            scope.currentNode = scope.root;
            return;
        }

        // we found a result but there wasn't a callback
        // which means we need to search deeper in the tree
        scope.currentNode = node;
    };
}
```
Here we're building a key press buffer and we're traversing the tree to look for matches. If we find a match that has a callback, it means we're at a leaf node and we execute the callback and reset the current node. If we do get a match, but there is no callback defined we continue traversing the tree until we find one. If nothing is found, we reset the current node back to the root. Also notice that the buffer is sorted and joined with a `+`, this is because the only reason we store anything in the buffer is for when a user pressed a combination of keys and holds them. Upon a `keyup` we clear the buffer.

Here is the `keyup` listener:

```typescript
function createKeyupListener() {
    return (e: KeyboardEvent) => {
        // remove the key from the buffer (a user might still hold other keys pressed)
        buffer = buffer.filter((x) => x !== normalizeKey(e.key));

        const scope: HotkeyScope = scopeMap.get(currentScopeName);

        if (scope && buffer.length === 0) {
            // no keys are held, reset the current node back to the root
            scope.currentNode = scope.root;
        }
    };
}
```
It's pretty straightward, really. We clear the buffer of the key that is being lifted and check if the buffer is empty, if it is, then we reset the current node back to the root.

A few other minor things we need before we wrap things up are a way for the user to get/set the current scope as well as to keep track if the key up/down event listeners are already attached to the DOM, so that we prevent multiple listeners attached, this way we keep things efficient:

```typescript
export const setHotkeysScope = (name: string) => {
    // No need to create a scope if there are no listeners
    // we will do the scope creation in the register function
    currentScopeName = name;
};

export const getHotkeysScope = () => currentScopeName;

let hasBoundKeyListenerToDom = false;
```

And finally, let's put everything together. Here is the full implementation:

```typescript
export function registerHotkey(
    hotkeys: string,
    scopeName: string,
    callback: (e: KeyboardEvent) => void,
    debounceTimeInMilliseconds = 1500
) {
    const scope: HotkeyScope = createScopeIfNeeded(scopeName);
    const processedHotkeys: string[] = hotkeys.split(" ").map((x) =>
        x
            .split("+")
            .map((y) => normalizeKey(y))
            .sort()
            .join("+")
    );

    const keydownListener = createKeydownListener(debounceTimeInMilliseconds);
    const keyupListener = createKeyupListener();

    const bind = () => {
        insertIntoTree(scope, processedHotkeys, callback);

        if (!hasBoundKeyListenerToDom) {
            document.addEventListener("keydown", keydownListener);
            document.addEventListener("keyup", keyupListener);

            hasBoundKeyListenerToDom = true;
        }
    };

    const unbind = () => {
        removeFromTree(scope, processedHotkeys);

        if (hasBoundKeyListenerToDom) {
            document.removeEventListener("keydown", keydownListener);
            document.removeEventListener("keyup", keyupListener);

            hasBoundKeyListenerToDom = false;
        }
    };

    // autobind
    bind();

    return { bind, unbind };
}
```
I added an optional parameter for defining different debounce timing before resetting the current node back to root. We create the key up/down key listeners and attach them to the DOM. Also we use the `hasBoundKeyListenerToDom` variable that we defined earlier to prevent us from registering the same listener multiple times.

One limitation of this implementation is that sequences take precedence over combinations, so for example if we have `a b` defined, we can't define a combination starting with `a` such as `a+c`, because we always find a match for the key that is currently pressed in the tree. This works really well for my project, but it's a limitation nevertheless.

I hope this post/tutorial was useful, I had a lot of time writing this code and I thought I'd share. At some point I will open source this and package it into its own library when I get the time to write some documentation and tests for it.

