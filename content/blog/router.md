---
external: false
draft: false 
title: The Router 
description: Understanding Stimulus Router
date: 2022-11-06
---
In the first post, we initialized the Stimulus Application to get an overview of its structure. Now we'll go
into more detail in its Router class, how it's initialized and how it interacts with the other pieces of the
puzzle.

As a recap from the previous chapter, this is how the Application class is instanciated in our site:
```
const application = Application.start()
```

this will trigger this static method, which will instanciate the Application:
```
  static start(element?: Element, schema?: Schema): Application {
    const application = new this(element, schema) // (1)
```
(1) the constructor is called with undefined values for element as schema, so it will use the default values. 

The Application class looks like this:
```
export class Application implements ErrorHandler {
...
  constructor(element: Element = document.documentElement, schema: Schema = defaultSchema) {
    this.router = new Router(this)
    ...
  }
...
}
```
So, the starting point is the application instance, this will create a Router with which 
it will be able to communicate. By the name itself, **Router**, we can expect this class to
be in charge of orchestrating the logic of the different events and objects existing in 
our application.



This is how the Router's constructor looks like:

```
export class Router implements ScopeObserverDelegate {
  readonly application: Application
  private scopeObserver: ScopeObserver
  private scopesByIdentifier: Multimap<string, Scope>
  private modulesByIdentifier: Map<string, Module>

  constructor(application: Application) { // (1)
    this.application = application
    this.scopeObserver = new ScopeObserver(this.element, this.schema, this) // (2)
    this.scopesByIdentifier = new Multimap() // (3)
    this.modulesByIdentifier = new Map() // (4)
  }
```

(1) In order to initialize the Router class, we only need to pass the application instance.
(2) A scopeObserver is created, this will be in charge of **TODO**.
(2) This will keep track of **TODO**.
(3) This will keep track of the **TODO**.

We're passing ```this.element``` which is a method 
on the Router itself. It reads the value of ``application.element``, so that's the HTML 
document (``document.documentElement``), the same logic applies to the second argument ``this.schema``, 
and the last argument will be the Router instance itself. Let's see how the ScopeObserver 
is constructed with those arguments: 
```
export class ScopeObserver implements ValueListObserverDelegate<Scope> {
  readonly element: Element
  readonly schema: Schema
  private delegate: ScopeObserverDelegate
  private valueListObserver: ValueListObserver<Scope>
  private scopesByIdentifierByElement: WeakMap<Element, Map<string, Scope>>
  private scopeReferenceCounts: WeakMap<Scope, number>

  constructor(element: Element, schema: Schema, delegate: ScopeObserverDelegate) {
    this.element = element
    this.schema = schema
    this.delegate = delegate // (1)
    this.valueListObserver = new ValueListObserver(this.element, this.controllerAttribute, this) // (2)
    this.scopesByIdentifierByElement = new WeakMap() // (3)
    this.scopeReferenceCounts = new WeakMap() // (4)
  }
```
(1) 
(2) A ValueListObserver will monitor changes related to 'data-controller' 
