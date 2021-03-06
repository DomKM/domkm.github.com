---
category: 100-days-of-hacking
layout: post
title: Isomorphic Clojure[Script]
subtitle: Part 1
description: "An experiment in applying isomorphic concepts to Clojure[Script] applications."
tags: [isomorphic, clojure, clojurescript, react, om, sente, nashorn]
comments: true
share: true
---

In pursuit of supporting a dynamic and interactive user experience, developers have been shifting from traditional server-rendered multi-page applications[^MPAs] (MPAs) to client-rendered single-page applications[^SPAs] (SPAs).

The web has matured and become a fairly capable application platform and SPAs utilize this platform to provide better experiences to users and developers. For users, applications can be more responsive to interaction and better facilitate dynamic experiences that are not possible/feasible with MPAs. For developers, building SPAs can provide a clearer separation of concerns and tends to enforce more client-side code structure. However, embracing SPAs is not without its drawbacks.

[^MPAs]: An MPA is a website that serves fully-formed or mostly-formed HTML when a client requests a page. Links cause the browser to request a new page; blowing away the previous DOM, CSSOM, and JS context. If rendered without JS execution, MPAs remain viewable and at least partially usable.

[^SPAs]: An SPA is a website that serves mostly bare HTML. Upon initialization, the JS requests data for that page and updates the DOM. Clicking links trigger asynchronous requests for new data and then modify the DOM (and sometimes `window.history` or `window.location`). The same DOM, CSSOM, and JS contexts live through the entire client session.


### MPA Pros & SPA Cons

* **SEO** \\
  Until [recently](http://googlewebmastercentral.blogspot.com/2014/05/understanding-web-pages-better.html), no major search engine executed JS. Though the only search engine that you care about reports to now execute JS, I would not be the first to question if this will put SPAs on equal footing with MPAs. Even if Google treats them the same, SPAs have significantly slower initial page-load speed and that negatively affects SEO. To serve fully-formed HTML, developers have resorted to crazy workarounds like rendering SPAs in headless browsers and serving HTML extracted from there. Needless to say, this introduces a whole new level of complexity.
* **Page-Load Performance** \\
  Serving fully-formed HTML is *fast*. Speed matters. Metrics from large companies have repeatedly demonstrated that user engagement decreases as latency increases. Rendering an initial page for an MPA requires fewer requests than for an SPA. For initial renders, SPAs additionally need to request and receive JS, execute it, and then request and receive data specific to that page.
* **Connection Quality** \\
  Slow or inconsistent connections can make SPAs unusable. This is particularly important for mobile users.
* **Accessibility** \\
  Most screen readers and "read it later" applications don't execute JS. The award for worst use of an SPA has to go to Google's Blogger; an application ostensibly meant to serve mostly static documents that is now slower than when it was an MPA and unusable without JS.
* **Graceful Degradation** \\
  MPAs tend to degrade more gracefully than SPAs. You're stuck with a blank page if JS fails to load on an SPA page, but you can probably still view the page and click links if it fails to load on an MPA page.
* **Maintainability** \\
  Validation, routing, and other logic often need to be shared by the client-side and server-side code. However, the codebases are typically written in different languages, so logic needs to be duplicated and then kept in sync.


### (Multi|Single)-Page Applications

We want a hybrid approach that blends the benefits of MPAs and SPAs. We want to serve fully-formed HTML for initial requests and, after the page loads, we want the client-side application to take over and become an SPA. We want to avoid the above issues and enjoy all of the benefits of SPAs without paying the costs traditionally incurred by them.

Isomorphic is a term used to describe applications where the frontend and backend code are written in the same language and share a significant amount of logic, which facilitates features like this. Isomorphism has primarily been achieved by using JS[^or-something-like-js] on the server. In fact, I've only seen the term "Isomorphic" used when immediately preceding "JavaScript," but that doesn't have to be the case.

[^or-something-like-js]: Or a language like CoffeeScript that is little more than syntactic sugar for JavaScript.

### A Better Language

ClojureScript tools have changed dramatically in the past year.

* [React](https://github.com/facebook/react), an excellent JS library for rendering, was released in May 2013. React supports multiple render targets; it can render to a DOM element or to an HTML string.
* [Om](https://github.com/swannodette/om), a ClojureScript wrapper for React that provides some important performance and architectural advantages, was released in January 2014.
* JDK 8, which ships with a high-performance JavaScript engine called Nashorn that is compatible with React, was released in March 2014.
* [core.async](https://github.com/clojure/core.async/), a Clojure[Script] library that provides a great concurrency model, was released in June 2013.
* [Sente](https://github.com/ptaoussanis/sente/), a Clojure[Script] library for bidirectional asynchronous communication over HTTP and Web Sockets via core.async's channel interface, was released in February 2014.

Though the toolset is not yet complete (which I will discuss further in Part 2), these libraries go a long way toward enabling isomorphism where it was previously infeasible.

Instead of running JS across the entire stack, we could run server-side Clojure, client-side ClojureScript, render fully-formed HTML using Nashorn with Om/React, and share client and server code since Clojure and ClojureScript have nearly identical syntax and semantics.

### Is Clojure[Script] Ready?

I wrote [Omelette](https://github.com/DomKM/omelette), an example application, to experiment with isomorphic Clojure[Script] patterns. It's a basic word-finding application. Given a string and desired position/s, it returns a list of words where the string is in the desired position. For example, searching for words that start with "om" would return a list that includes "omelette."

The goal was to make an isomorphic SPA where all routes are renderable by the server without a significant amount of extra work, code duplication, or `(if server? ...)` conditionals. Additionally, the client-side code should be able to take over transparently and quickly; no white flash and no additional requests.

### Abstractions

The code needed to work in three dissimilar environments: Clojure in the JVM, ClojureScript in Nashorn in the JVM, and ClojureScript in the browser. Sharing code requires abstractions.

#### State Management

The state of the application needs to be able to be usable in every environment. This one is easy since Clojure and ClojureScript have nearly identical syntax and semantics and can serialize to and deserialize from [EDN](https://github.com/edn-format/edn).

Omelette stores application state as a 2-tuple where the elements are a namespaced page keyword and a map of page data. For example, an application state of `[:omelette.page/search {:query "bloop" :options #{:prefix}}]` represents the "Search" page with a query input value of `"bloop"` and the `:prefix` option checked, while `[:omelette.page/about {:markdown "# About ..."}]` represents the "About" page with Markdown content.

#### Rendering

The application needs to be able to render to a DOM element and to an HTML string on the client-side and server-side respectively. Also, the client-side application needs to be able to transition from static, server-rendered HTML to dynamic DOM rendering when it initializes in a browser.

Omelette uses Om/React for rendering. To serve HTML, Omelette server-side code uses an application state 2-tuple to render the view to a string within Nashorn, which returns a normal instance of `java.lang.String`. The HTML is embedded in a `div` in the document layout. The state is also serialized as EDN and embedded in the layout so that, when Omelette client-side code initializes in a browser, it doesn't have to make an additional request to fetch the initial data.

#### Data Retrieval and Persistence

Retrieving and persisting data needs to be easy to do for both initial page requests and running client-side applications.

Omelette uses Sente for communication and reuses Sente's handler function for server-side rendering. Using the same handler makes it very easy to share this functionality.

#### Routing

Routing needs to work without requiring access to environment-specific things like the browser's `window` object or the server's incoming HTTP request. However, routes should be *able* to make use of environment-specific things, particularly on the server for authentication and authorization.

Omelette's search page is rendered at `"/"`, `"/search"`, and `"/search/:options/:query"` where `:options` is the desired position/s and `:query` is the target string. For example, `"/search/prefix/bloop"` is the path to find words that start with `"bloop"`. The "About" page is rendered at `"/about"`.

Routing in Omelette is unlike any routing functionality that I have seen elsewhere, so we should probably discuss it here. It has two steps:

1. A path (`"/search/bloop/prefix"`) is transformed into an application state 2-tuple (`[:omelette.page/search {:query "bloop" :options #{:prefix}}]`).
2. An application state 2-tuple (`[:omelette.page/search {:query "bloop" :options #{:prefix}}]`) is passed through a handler function that pattern matches the state using [core.match](https://github.com/clojure/core.match) and returns or responds with a modified state (`[:omelette.page/search {:query "bloop" :options #{:prefix} :results ("bloop" "blooper" "blooping")}]`)

All routes and application states are bidirectional. For example, the path `"/search/bloop/prefix"` can be derived from the application state returned from either of the above steps.

Additionally, though this is not strictly routing, document titles can be derived from application states. This is used for setting and updating the document title on the server and client respectively.

Routing in Omelette is entirely ad hoc. In other words, routes are not defined using any sort of abstraction, but are manually converted to and from application state 2-tuples.

Routing is not a solved problem. There are no major Clojure[Script] routing libraries.

### Step by Step

1. A request is made to `"/search/prefix/bloop"`.
2. The request is converted to application state 2-tuple: `[:omelette.page/search {:query "bloop" :options #{:prefix}}]`.
3. The state gets passed through a handler function that returns a complete state: `[:omelette.page/search {:query "bloop" :options #{:prefix} :results ("bloop" "blooper" "blooping")}]`.
4. The complete state is serialized to EDN and passed to Om/React through Nashorn, which returns an HTML string for that state.
5. The Om/React HTML string and application state EDN are inserted into the `body` of an HTML layout to form a complete document.
6. A response is sent with the document and a status that is determined by the page keyword; 404 if `:omelette.page/not-found`, 200 otherwise.
7. The Om/React application loads, uses the embedded EDN as the initial application state, and then mounts into the embedded HTML.
8. The ClojureScript router listens for navigation events and application state changes.
9. When the ClojureScript router detects a change, it updates `window.history` and sends the new application state (either the authoritative state or a state derived from the navigation event) to the server via Sente.
10. Sente passes the new state to the Clojure handler function. Since this time it is not an initial request, the router responds via Sente with an updated state.
11. Sente passes the new new state to the ClojureScript handler function which then updates the application state.
12. GOTO 9

### Wrapping Up

The full code for Omelette: [github.com/DomKM/omelette](https://github.com/DomKM/omelette)

Take a look at [`omelette.render` to see how Nashorn is used](https://github.com/DomKM/omelette/blob/master/src%2Fomelette%2Frender.clj) and [`omelette.view` to read through the Om code for rendering](https://github.com/DomKM/omelette/blob/master/src%2Fomelette%2Fview.cljs). The bulk of Omelette's code is in [`omelette.route`](https://github.com/DomKM/omelette/blob/master/src%2Fomelette%2Froute.cljx) This uses [cljx](https://github.com/lynaghk/cljx), a tool that compiles .cljx files to both .clj and .cljs files. Forms prepended with `#+clj` and `#+cljs` will only be output to .clj and .cljs files respectively. This makes it easy to target both Clojure and ClojureScript with the same codebase. `omelette.route` provides routing helper functions, a ClojureScript router component using Om, and a Clojure router component using the descriptively but generically named [Component](https://github.com/stuartsierra/component).

Part 2 will discuss problems with the above approach to isomorphic Clojure[Script] and new libraries/patterns to alleviate these problems. Spoiler: It will be feasible to do server-side rendering without using Nashorn or any JS engine in the not-too-distant future.


*Thanks to [Nelson Morris](http://nelsonmorris.net/) and [Kyle Kingsbury](http://aphyr.com/) for reviewing drafts of Omelette.*
