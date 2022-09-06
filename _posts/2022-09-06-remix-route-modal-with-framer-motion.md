---
layout: post
title: "Remix route modal with Framer Motion "
description: "Building a modal with a route and enter and exit animations using Remix and Framer Motion (and a bit of Tailwindcss)"
date: 2022-09-06
tags: [programming, react, remix-run, framer-motion, tailwindcss, typescript, javascript, tutorial]
comments: false
share: true
---

I'm working on a [Remix](https://remix.run/) app and I wanted to add a modal with its own route. I was already using [Framer Motion](https://www.framer.com/motion/) for bunch of animations in the project and I wanted to use it for the modals (opening and closing animations) as well.

I found a few discussions in the Remix discord and GitHub discussions, but most of the solutions didn't provide a way for the exit animations to trigger and complete before closing the modal.

So this is what I did.

First I'll show you the modal (I've omitted parts of the code that are not relevant for brevity, link to the full source code in the end of the post):


{% raw %}
```tsx
// /app/components/Modal.tsx
export default function Modal(
    props: React.PropsWithChildren<{
        title?: string;
        isDisabled?: boolean;
        onDismiss: () => void;
    }>
) {
    // ...bunch of code that is omitted handling Esc, UI jittering, clicking outside etc.

    const { onDismiss, isDisabled } = props;

    function handleDismissClick() {
        // You should use this function also for pressing esc
        onDismiss();
    }

    return (
        <>
            {createPortal(
                <>
                    {/* Overlay */}
                    <div className="fixed inset-0 z-30">
                        <motion.div
                            initial={{ opacity: 0 }}
                            animate={{ opacity: 0.75 }}
                            exit={{ opacity: 0 }}
                            transition={{ duration: 0.15 }}
                            className="absolute inset-0 bg-zinc-700"
                        />
                    </div>

                    {/* Modal */}
                    <motion.div
                        initial={{ opacity: 0, scale: 0.95 }}
                        animate={{ opacity: 1, scale: 1 }}
                        exit={{ opacity: 0, scale: 0.95 }}
                        transition={{ duration: 0.15 }}
                        className="fixed inset-0 z-40 flex items-center justify-center"
                        role="dialog"
                        aria-modal="true"
                        aria-labelledby="modal-title"
                    >
                        <div
                            className="z-50 w-full overflow-hidden rounded bg-white text-zinc-900 shadow-md dark:bg-zinc-800 dark:text-zinc-200 sm:max-w-lg"
                        >
                            <div className="flex items-center border-b px-5 py-4 dark:border-zinc-700">
                                <h3
                                    className="flex-1 text-lg font-bold leading-6 tracking-tight"
                                    id="modal-title"
                                >
                                    {props.title}
                                </h3>
                                {!isDisabled && (
                                    <button
                                        onClick={handleDismissClick}
                                        className="text-zinc-500 outline-purple-500 hover:text-purple-500"
                                    >
                                        <XMarkIcon className="h-4 w-4" />
                                    </button>
                                )}
                            </div>

                            <div>{props.children}</div>
                        </div>
                    </motion.div>
                </>,
                document.getElementById("modal-container") as Element
            )}
        </>
    );
}
```
{% endraw %}

That's a lot of code, but it's mostly styling ([Tailwindcss](https://tailwindcss.com/) in this case) and structure, your modal could be simpler. The important thing to note are the `<motion.div>` components and their animation props, for both the overlay and the modal itself. For more info on those, check out the Framer Motion docs, but just very briefly you have the `initial` prop which basically defines what's the initial animation state, the `animate` prop which defines what the animation should transition to, and the `exit` prop which is the exit transition from the `animate` state to `exit`. You also have a `transition` prop in which you can define how long your animation duration should be.

So now that the modal is defined, here is the actual Remix stuff:

```tsx
// /app/routes/hello.tsx -> our /hello route
import { Link, Outlet } from "@remix-run/react";

export default function Index() {
    return (
        <div>
            <h1 className="text-3xl font-bold">So I heard you like modals?</h1>
            <div className="mt-4">
                <Link
                    to="/hello/edit"
                    replace={true}
                    className="underline text-purple-600 hover:text-purple-700"
                >
                    Open a Modal
                </Link>
            </div>
            <Outlet />
        </div>
    );
}
```

Here I have a Remix `Link` component that points to `/hello/edit`, I've also set the `replace` prop to `true` so that I don't polute my browser history, but that's really up to you and your scenario. In addition we have an `Outlet` component that will handle the modal as a nested route (for more info check the remix docs).

Next we have the `/hello/edit` route as follows:

```tsx
// /app/routes/hello/edit

import { useState } from "react";

import { useNavigate } from "@remix-run/react";
import { AnimatePresence } from "framer-motion";

// Our modal from earlier
import Modal from "~/components/Modal";

export default function Edit() {
    const [isModalOpen, setIsModalOpen] = useState(true);

    const navigate = useNavigate();

    function handleDismiss() {
        setIsModalOpen(false);
    }

    function handleExitComplete() {
        navigate("/hello", { replace: true });
    }

    return (
        <AnimatePresence onExitComplete={handleExitComplete}>
            {isModalOpen && (
                <Modal title="My modal" onDismiss={handleDismiss}>
                    <div className="space-y-4 px-5 py-4">Check out the url</div>
                </Modal>
            )}
        </AnimatePresence>
    );
}
```

The important bits here are the `AnimatePresence`, this is a Framer Motion component that allows components to animate out when they're removed from the React tree. We also set its `onExitComplete` prop, which will trigger once the component within `AnimatePresence` is removed, we then navigate back to `/hello` (the parent route) and again prevent storing this navigation in the browser history.

That's it, not we have a modal with its own route and both enter and exit animations.

You can find the full source code of this example [here](https://github.com/n1ghtmare/remix-modal-example).
