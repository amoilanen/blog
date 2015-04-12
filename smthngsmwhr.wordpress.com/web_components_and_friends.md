[Web App Development Challenges](#web-app-development-challenges)
[Frameworks, libraries, standards](#frameworks-libraries-standards)
[Need of Componentization](#need-of-componentization)
[Web Components](#web-components)
[Polymer](#polymer)
[React.js](#react-js)
[Angular.js](#angular-js)



# <a name="web-app-development-challenges"></a> Web App Development Challenges

Web applications become ever larger, sophisticated and virtually indistinguishable from desktop apps. In ["JavaScript and Friends: CoffeeScript, Dart and TypeScript"](https://smthngsmwhr.wordpress.com/2013/02/25/javascript-and-friends-coffeescript-dart-and-typescript/) we discussed how JavaScript is adapting to this trend and what alternative languages appeared that trying to address some of the pain points.

There are multiple challenges in developing large modern Web apps, most important of them are listed below.

  * **Maintainability**

    It should be relatively easy to find a piece of code responsible for a certain part of the app. Ideally this piece code should encapsulate some of the implementation details of that part and be easy to understand and modify without introducing any bugs.

  * **Testability**

    It should be easy to instantiate pieces of code comprising the app in a test environment and verify their behavior. We would also like to test how those pieces interact with each other and not just that every one of them works on its own.

  * **Code reuse**

    If in different areas of the application there are similar UI elements and functionality (such as, for example, dialogs) it should be possible to reuse the same code, markup and styles.

  * **View updates**

    When the data being presented through the UI as part of some view is modified, the corresponding view needs to be re-rendered. We should be smart about re-rendering only the parts of the app that really need it and, for example, avoid loosing the visual context after a complete re-render. 

These challenges are more or less successfully resolved by various libraries and frameworks or combinations of them. One notable example many are familiar with is Angular.js which to certain extent addresses them all.

However, in this post we are not going to discuss the intricate details of frameworks like Angular.js, partly because they offer much more than just componentization and code reuse, and also in order to stay a bit more focused and fit the limitations of a single article. Neither are we really going to particularly discuss other challenges besides componentization as every one of them would merit a separate blog post. For example, it is more or less established that some form of MVC (MVV, MVP, etc.) can help with maintainability and view updates, but we will not discuss this here. Fully covering the outlined challenges and different approaches to address them will probably lead to a medium sized book on developing Web applications. 

So we will try to stay focused on one particular topic: componentization and discuss some of the philosophy and ideas that various frameworks and libraries embrace in this area but which are unfortunately often lost behind the intricate technical details of APIs they expose or are not emphasized enough and underutilized by developers. For example, I saw at least a few Angular projects that did not reuse as much code as they could have just because they for some reason underutilized the use of custom directives (a bit later we will discuss how those relate to Web Components). After reading this article hopefully you will come out with a better grasp of the subject and also understand that despite superficial differences between APIs some common principles behind are quite similar.

# <a name="frameworks-libraries-standards"></a>Frameworks, libraries, standards

In this post we will be talking about, looking into the technical details and comparing Web Components, Polymer, React.js and Angular.js to see how they relate to each other in regard to componentization, and what similarities and differences there are. But before we can do that, we first should establish some initial relationship between this set of technologies and briefly explain what every one of them is and is not. This will simplify further understanding and comparisons.

 * **Web Components** http://webcomponents.org/

 Is just a set of standards natively supported by browsers that facilitate development of reusable components. The level of support is different depending on the browser. This is not a library or framework.

 * **React.js** https://facebook.github.io/react/

 Is a functional-style "view" library for feeding data into a tree of React components and rendering this tree into HTML. This is not a framework as React does not suggest an expected way of creating a web app and just deals with the view part. In fact you can use React with other frameworks, but the most canonical way suggested by Facebook would be to use it in the Flux architecture.

 * **Polymer** https://www.polymer-project.org/

 Is a polyfill library to bring Web Components standards to different browsers that do not support it natively yet. In addition to implementing the current version of the standards Polymer adds a few things of its own that are not yet in the standards and may be even never included there. So we can say that Polymer is a library based on the Web Components standards, but like React it is also not a framework.

 * **Angular.js** https://angularjs.org/

 Unlike the previous three, this is a true opinionated framework that to a large extent dictates (for better or for worse, depends on your viewpoint) how a Web app should be structured and in what exact way certain things should be done. There is still some flexibility but most of the time you just should do things Angular way. It also allows to create reusable components, but those components will be reusable only in another Angular.js app (like those made with React, more discussion on the topic of interoperability later in the article).

Of course we can cover other frameworks and libraries and even venture to create some site similar to http://todomvc.com/ but this time not for comparing the MVC features but support for componentization. If somebody feels like doing such a site, feel free to fork and extend https://github.com/antivanov/ui-components that contains examples that we will use in this article. We will, however, focus only on those libraries/frameworks listed above as in general 3-4 different examples should already give a good feeling of the way things are done and have some meaningful discussion and comparison.

# <a name="need-of-componentization"></a>Need of Componentization

You might have heard of domain-specific languages (DSLs) http://martinfowler.com/books/dsl.html and such before in relation to other areas of software development and not necessarily front-end or Web development. Feel free to read more on the subject in case you are interested, although this is not required to understand the present article. In brief the main idea is that in order to solve some challenging problem it often pays off to first create a language in which this problem is then easily solvable. Then having this language will allow you to describe the system and its behavior much easier. One good example would be SQL to manipulate and query data stored in a relational database. Try doing that in your plain code and you will be a world of pain and maintainability nightmare if your data schema and queries are complex enough.

The idea is that as Web applications become more complex, they are in many respects not that different from other areas of Software Development and can certainly benefit from creating DSLs. In fact HTML is already such a DSL albeit quite a low level and general purpose one. HTML5 introduces some new semantic tags such as &lt;section&gt;, &lt;header&gt;, &lt;footer&gt;, etc. but this still is not high enough level of abstraction if you develop a large app. What we would like to have is to operate with more high things, in the case of a Web store this might be: &lt;cart&gt;, &lt;searchbox&gt;, &lt;recommendation&gt;, &lt;price&gt;, etc.

Of course if your app is quite simple and is just a couple of pages you may not need so much componentization and code reuse. Compare this to the case when you need to write some simple script in, say, Python, then you may not need to define any classes or even functions in that script. Still, it is nice to have the ability to create reusable abstractions (functions, classes) when there is a genuine need for them. Components in a Web app are not all that different from functions or classes in this regard.

The obvious benefits of creating and using components are better maintainability, code reuse and speed of development once you already have the necessary components developed. Components are like building blocks from which you can build your app, if those blocks are well-designed you can just quickly throw together some functionality that before would require a lot of boiler-plate coding. Same as with functions and classes, but just on a bit different level.

Hopefully, by now you are convinced in the potential usefulness of components and would like to know more about the technical details. So far our discussion has been a bit abstract and philosophical, it is right about time to delve into examples and see how these ideas are used in practice.

#### **Examples**

We chose to implement a simple Breadcrumbs component using different technologies listed above on this page with examples http://antivanov.github.io/ui-components

The source code is available here https://github.com/antivanov/ui-components, feel free to check out and modify it, and we will go deeper into various frameworks and libraries in the next sections, as well as will cover some of the code in the examples.

# <a name="web-components"></a>Web Components

We will start with Web Components, a set of evolving standards and best practices that allow to create and reuse HTML elements. Let's quickly go over some of the standards. The example code https://github.com/antivanov/ui-components/tree/master/WebComponents/breadcrumbs

#### **HTML imports**

Allows to include HTML documents into other HTML document. In our example we include <b>breadcrumbs.html</b> into <b>breadcrumbs.demo.html</b> by inserting the following import into the latter:

```html
<link rel="import" href="breadcrumbs.html">
```

That's pretty much all there is to it, there are, of course, some details such as how we avoid including the same file twice, detecting circular references, etc. read more here http://webcomponents.org/articles/introduction-to-html-imports/

#### **Custom elements**

Our Breadcrumbs component is implemented as a custom HTML element, that we register as follows:

```javascript
    document.registerElement('comp-breadcrumbs', {
      prototype: prototype
    });
```

where <b>prototype</b> is some object which defines what methods and fields will be available on a Breadcrumbs instance that we create like this:

```javascript
var breadcrumbsElement = document.createElement('comp-breadcrumbs');
```

We also could have just used this new custom element in the HTML markup and it would have been instantiated just like the rest of HTML elements, more examples http://webcomponents.org/tags/custom-elements/

#### **Shadow DOM**

Custom element can have shadow DOM associated with it. The elements belonging to this shadow DOM are encapsulated inside this custom element and are hidden and considered to be implementation details of the custom element. Some predefined HTML elements have shadow DOM as well, for example &lt;video&gt;. When we want the Breadcrumbs element to have associated shadow DOM, we create a shadow root and add a DOM element to it:

```javascript
this.createShadowRoot().appendChild(element);
```
Shadow DOM allows to avoid revealing component implementation details on the HTML level to other components and client code using this component. This results in the HTML being more modular and high level. More on Shadow DOM http://webcomponents.org/articles/introduction-to-shadow-dom/

#### **HTML templates**

Finally it is possible to use templates, a special &lt;template&gt; tag is introduced for that purpose. In our example in &lt;breadcrumbs.html&gt;:

```html
<template id="breadcrumbs-template">
  <style>
    ...
    .crumb {
      border: 1px solid transparent;
      border-radius: 4px;
    }
    ...
  </style>
  <div class="breadcrumbs">
  </div>
</template>
```

Our template also includes some styling information that will be applied only to the HTML created from this template. In this manner templates also enable hiding CSS implementation details and provide for better encapsulation. The contents of the template are not active until a node has been created from this template: images will not be fetched, scripts will not be executed, etc. There is no data binding support, template is just a piece of static HTML.

In order to create an element from template, we first should import template DOM node into the current document:

```javascript
var element = document.importNode(template.content, true);
```

More details http://webcomponents.org/articles/introduction-to-template-element/

#### **Breadcrumbs example**

The Breadcrumbs example demonstrates how these different standards can be used together to create a reusable component which, as we noted, will just be a custom element. Let's look at the code of &lt;breadcrumbs.html&gt;:

```javascript
<template id="breadcrumbs-template">
  <style>
    .crumb,
    .crumb-separator {
      padding: 4px;
      cursor: default;
    }
    .crumb {
      border: 1px solid transparent;
      border-radius: 4px;
    }
    .crumb:hover,
    .crumb:focus {
      background-color: #f2f2f2;
      border: 1px solid #d4d4d4;
    }
    .crumb:active {
      background-color: #e9e9e9;
      border: 1px solid #d4d4d4;
    }
    .crumb:last-child {
      background-color: #d4d4d4;
      border: 1px solid #d4d4d4;
    }
  </style>
  <div class="breadcrumbs">
  </div>
</template>
<script>
  (function() {

    function activateCrumb(self, crumb) {
      var idx = parseInt(crumb.getAttribute('idx'));
      var newPath = self.path.slice(0, idx + 1);

      if (newPath.join('/') != self.path.join('/')) {
        var event = new CustomEvent('pathChange', {
          'detail': newPath
        });
        self.dispatchEvent(event);
      }
    }

    function renderPath(self, path) {
      var maxEntries = parseInt(self.getAttribute('maxEntries')) || -1;
      var renderedDotsSeparator = false;

      while(self.container.firstChild) {
        self.container.removeChild(self.container.firstChild);
      }
      path.forEach(function(pathPart, idx) {

        //Skip path entries in the middle
        if ((maxEntries >= 1) && (idx >= maxEntries - 1) 
          && (idx < path.length - 1)) {

          //Render the dots separator once
          if (!renderedDotsSeparator) {
            self.container.appendChild(
              createDotsSeparator(path, maxEntries)
            );
            self.container.appendChild(createCrumbSeparator());
            renderedDotsSeparator = true;
          }
          return;
        }

        self.container.appendChild(createCrumb(pathPart, idx));
        if (idx != path.length - 1) {
          self.container.appendChild(createCrumbSeparator());
        }
      });
    }

    function createDotsSeparator(path, maxEntries) {
      var crumbSeparator = document.createElement('span');
      var tooltipParts = path.slice(maxEntries - 1);

      tooltipParts.pop();

      var tooltip = tooltipParts.join(' > ');

      crumbSeparator.appendChild(document.createTextNode('...'));
      crumbSeparator.setAttribute('class', 'crumb-separator');
      crumbSeparator.setAttribute('title', tooltip);
      return crumbSeparator;
    }

    function createCrumb(pathPart, idx) {
      var crumb = document.createElement('span');

      crumb.setAttribute('class', 'crumb');
      crumb.setAttribute('tabindex', '0');
      crumb.setAttribute('idx', idx);
      crumb.appendChild(document.createTextNode(pathPart));
      return crumb;
    }

    function createCrumbSeparator() {
      var crumbSeparator = document.createElement('span');

      crumbSeparator.appendChild(document.createTextNode('>'));
      crumbSeparator.setAttribute('class', 'crumb-separator');
      return crumbSeparator;
    }

    var ownerDocument = document.currentScript.ownerDocument;
    var template = ownerDocument.querySelector('#breadcrumbs-template');
    var prototype = Object.create(HTMLElement.prototype);

    prototype.createdCallback = function() {
      var self = this;
      var element = document.importNode(template.content, true);

      //Current path
      this.path = [];

      //Crumbs container
      this.container = element.querySelector('.breadcrumbs');
      this.container.addEventListener('click', function(event) {
        if (event.target.getAttribute('class') === 'crumb') {
          activateCrumb(self, event.target);
        }
      }, false);
      this.container.addEventListener('keypress', function(event) {
        if ((event.target.getAttribute('class') === 'crumb') 
            && (event.which == 13)) {
          activateCrumb(self, event.target);
        }
      }, false);
      this.createShadowRoot().appendChild(element);
    };
    prototype.setPath = function(path) {
      this.path = path;
      renderPath(this, path);
    };
    document.registerElement('comp-breadcrumbs', {
      prototype: prototype
    });
  })();
</script>
```

In lines *1-28* we define the template and CSS styles that will be used for the component.
The JavaScript code of the component is contained in the &lt;script&gt; tags and we register the custom element *comp-breadcrumbs* in lines *137-139*. As an argument we pass a prototype object that contains fields and methods that will be attached to a new element instance once it is created. Let's go over these methods in more detail.

*133-136* the component exposes method *setPath* that can be called by an external code directly on the DOM *comp-breadcrumbs* element, we just remember the path passed as an argument and re-render the component.

*107-132* is the most interesting part as we define the *createdCallback* which is a hook method that is called when a custom element is created. In lines *107-108* we do some hoops to get the template we just defined, it is a bit tricky since the current document when we load *breadcrumbs.html* is that of the demo HTML *breadcrumbs.demo.html*. *109* we inherit from *HTMLElement*. *113* import the node to the current document and in line *131* create shadow root and add the imported template to it. The lines *116-130* attach some listeners and define initial values for fields.

*44-105* deal with rendering the currently set path as a set of crumbs, crumb separators and dots to the shadow DOM of the current instance of *comp-breadcrumbs*. We do not use any libraries like jQuery which might have shortened some of the DOM element creation boilerplate. In line *45* we check the value of the *maxEntries* attribute and if it is set we make sure that we render dots instead of extra breadcrumbs.

Note how we have to handle data-binding all by ourselves and make sure that the changes in the state of the component are properly reflected in the generated HTML (view). This feels a bit low-level and tedious, certainly we could use some library or framework for this task, but more on this later. 

*32-42* we define a handler which is triggered whenever a particular crumb is activated, either by clicking on it or pressing *Enter*. In lines *37-40* we create and dispatch custom *pathChange* event so that the client code can listen for this event and act accordingly, for example, navigate to a certain page inside the app.

A very nice thing is that with our new custom element we can just use all the familiar DOM APIs and event system we already know: *getAttribute*, *createElement*, *querySelector*, *addEventListener*, etc. Then all the nuances of custom element implementation are well-hidden from its users and they can deal with it as if it were just like any other normal DOM element, which provides for excellent encapsulation and does not lock us into using some particular library or framework together with our custom elements.

#### **How to use in your project**

Some browsers do not yet support all of these standards and although Web Components is definitely the future way of creating reusable components, while it still evolves and is being adopted you have to use something else. The first option to consider would be Polymer, a library that brings Web Components support to all browsers and adds some of its own features on top. Arguably even in the future when all browsers support Web Components and the standards matured a lot you will still have to use some higher level library, because standards tend to stay quite generic and low level in order not to overburden the browser developers with more features to support and also leave some flexibility for library developers.

# <a name="polymer"></a>Polymer

One such library you may use to enable Web Components in your apps is Polymer https://www.polymer-project.org. It enables Web Components for browsers that do not yet support corresponding standards. For example, with minimal changes we can make our Breadcrumbs custom element work in Firefox in addition to Chrome. However, some things will still not work quite the way we would like them to, for example, since there is no shadow DOM implementation yet in Firefox the DOM elements generated by our custom element will be added directly to the DOM and will not be hidden from the client code. The example http://antivanov.github.io/ui-components/Polymer/breadcrumbs/breadcrumbs.demo.html, source code https://github.com/antivanov/ui-components/blob/master/Polymer/breadcrumbs/breadcrumbs.standard.only.html

Let's quickly go over a few changes we have to make. The parts common with the Web Components example are omitted for brevity's sake:

```javascript
<link rel="import" href="../bower_components/polymer/polymer.html">

<polymer-element name="comp-breadcrumbs" tabindex="0">

<template id="breadcrumbs-template">
  <style>
    ...
    .crumb {
      border: 1px solid transparent;
      border-radius: 4px;
    }
    ...
  </style>
  <div class="breadcrumbs">
  </div>
</template>
<script>
  (function() {
    ...
    var prototype = {};

    prototype.path = [];
    prototype.domReady = function() {
      var self = this;

      //Crumbs container
      this.container = this.shadowRoot.querySelector('.breadcrumbs');
      this.container.addEventListener('click', function(event) {
        if (event.target.getAttribute('class') === 'crumb') {
          activateCrumb(self, event.target);
        }
      }, false);
      this.container.addEventListener('keypress', function(event) {
        if ((event.target.getAttribute('class') === 'crumb') 
            && (event.which == 13)) {
          activateCrumb(self, event.target);
        }
      }, false);
      renderPath(this, this.path);
    };
    ...

    Polymer('comp-breadcrumbs', prototype);
  })();
</script>
</polymer-element>
```

In line *1* we import the Polymer library using familiar HTML import which is supported by Firefox. First difference with the purely Web Components based example is noticable in the lines *3* and *46*: we wrap the &lt;script&gt; and &lt;template&gt; definitions inside &lt;polymer-element&gt; for which we also specify *tabindex* attribute so that our custom element can be focused from keyboard using the *Tab* key.

In line *20* the custom element prototype is just an empty object and we do not have to inherit from *HTMLElement* like before. We cannot create a real shadow root in many browsers so the Polymer API differs here as well. In line *27* we just query for a fake "shadow root" and at this point our template has already been rendered and the DOM element appended to that root.

Finally in line *43* we register our new custom element with Polymer which also differs from the standard way of doing this.

Besides support for Web Components standards which are pretty low-level and generic Polymer also brings in some of its own higher-level features that might make it worse using even when Web Components support becomes very mainstream and common.

To illustrate some of those features let's quickly go over another example that also deals with the Breadcrumbs component https://github.com/antivanov/ui-components/blob/master/Polymer/breadcrumbs/breadcrumbs.html Most interesting parts are as before are where this example differs from the previous one:

```javascript
<link rel="import" href="../bower_components/polymer/polymer.html">

<polymer-element name="comp-breadcrumbs">

<template id="breadcrumbs-template">
  <style>
    ...
  </style>
  <div class="breadcrumbs" on-keypress="{{_onKeypress}}">
    <template id="crumbs" repeat="{{crumb, index in crumbs}}">
      <span class="{{crumb.dots ? 'crumb-separator' : 'crumb'}}" 
        idx="{{crumb.idx}}" title="{{crumb.tooltip}}" 
        on-click="{{_onActivateCrumb}}" 
        tabIndex="0" >{{crumb.value}}</span>
      <span class="crumb-separator">></span>
    </template>
  </div>
</template>
<script>
  (function() {
    ...

    var prototype = {};

    prototype.path = [];
    prototype.crumbs = [];
    prototype.setPath = function(path) {
      this.path = path;
      this.crumbs = renderPath(this, path);
    };
    ...

    Polymer('comp-breadcrumbs', prototype);
  })();
</script>
</polymer-element>
```

As can be seen the templating features of Polymer are more advanced than the standard Web Components ones. For example, in lines *9* and *13* we can bind event handlers to the HTML generated from the template directly in the template code.

Polymer templates are automatically bound to the state of the current component, for example, see line *14* where we refer to *crumb.value* pretty much like we would do if we used some template system like Mustache https://github.com/janl/mustache.js

Line *10* demonstrates how we can iterate over model with *repeat* when generating repetitive HTML markup, also a very nice feature that saves our effort from doing the same thing in JavaScript on our own as we saw in the previous example.

More information about Polymer templating https://www.polymer-project.org/0.5/docs/polymer/template.html

Another useful thing provided by Polymer which is not shown in this simple example is that it exposes in the form of components (well, what else would you expect?) quite a lot of useful functionality that we can reuse when developing our own components. One example of such a useful component made available to us by Polymer is *core-ajax* https://www.polymer-project.org/0.5/docs/elements/core-ajax.html

For example this Progress bar extends the *core-range* component https://github.com/Polymer/paper-progress/blob/master/paper-progress.html All we need to do to take advantage of an already existing Polymer implementation that we might need, is to include the *extends* attribute to the Polymer component declaration as described in the documentation https://www.polymer-project.org/0.5/docs/polymer/polymer.html#extending-other-elements

#### **Existing component libraries**

There are a few components (called "elements" in the Polymer terminology) developed with Polymer by somebody else that you can reuse on your own project. For example, take a look at http://customelements.io where everybody can publish their components. And also Polymer itself provides quite a few ready to use components right out of the box such as the ones that can be found here https://www.polymer-project.org/0.5/docs/elements/material.html

# <a name="react-js"></a>React.js

While the Web Components standards are still evolving, some libraries try to solve the problem of componentization and code reuse in Web apps in their own way.

One such library is React.js https://facebook.github.io/react/ Besides support for building reusable components it also tries to solve the view update problem *"React implements one-way reactive data flow which reduces boilerplate and is easier to reason about than traditional data binding"*. We will try to explain how React works, but if you wish to read more feel free to follow the tutorial https://facebook.github.io/react/docs/tutorial.html

Authors of React.js have made a few architectural choices that set it apart from other approaches we are discussing in this article.

First, everything is a component, and to build an app one actually has to first think how to split that app into components, how to compose and nest them. This is quite different from using Web Components where you can just optionally define components if you wish to do so, but you can use whatever low level HTML and JavaScript you wish together with it. In our example, even the demo app that uses the Breadcrumbs component we develop is a separate component https://github.com/antivanov/ui-components/blob/master/React.js/breadcrumbs/breadcrumbs.demo.js:

#### **How to define a component, rendering, events, properties and state**

```javascript
var BreadcrumbsDemo = React.createClass({
  getContent: function(path) {
    return path[path.length - 1];
  },
  getInitialState: function() {
    return {
      path: this.props.path
    };
  },
  onPathChange: function(value) {
    this.setState({
      path: value
    });
  },
  reset: function() {
    this.setState({
      path: this.props.path
    });
  },
  render: function() {
    return (
      <div>
        <div id="breadcrumb-container">
          <Breadcrumbs path={this.state.path} maxEntries="5" 
            onChange={this.onPathChange}/>
        </div>
        <div id="content">{this.getContent(this.state.path)}</div>
        <button id="resetButton" onClick={this.reset}>Reset</button>
      </div>
    )
  }
});

var fullPath = ['element1', 'element2', 'element3', 'element4',
  'element5', 'element6', 'element7'];

React.render(
  <BreadcrumbsDemo path={fullPath}/>,
  document.querySelector('#container')
);
```

The most important part of a component definition is method **render** in lines *20-31* where the component defines how it is rendered. This can include other components like *Breadcrumbs* in line *24* and familiar HTML elements like *div* in line *27*.

We can also parameterize the rendition with the component data which can come from two sources: component state and component properties.

*Properties* of a component should be considered immutable and are passed from a parent component, like it is done in the line *38* where we pass *fullPath* to be used with our *Breadcrumbs* component in the demo. As it can be seen every component has a parent or is a root-level component like our *BreadcrumbsDemo* component.

*State* of a component is something than can be changed during the component lifecycle. Looks like this is a necessary compromise in the design of React.js to still be able to do local changes to a component without propagating them all the way from the root component. However, beware of the state, it should be used very sparingly, and if something can be a property and used by several nested or sibling components, make it an immutable property. Method **getInitialState** in lines *5-8* is used to define what should be the default initial state of *BreadcrumbsDemo*. State can be accessed like in line *27* via the *state* property on the instance of the current component. Setting state is also simple, you just need to call the **setState** method as it is shown in lines *16-18*.

Note that whenever the state or properties of the current component are changed and they are used in the component rendition, then React will re-render component. This is how the *view update* development challenge is solved by React, it will update the view automatically once you declaratively described in the *render* method how it should be done.

In fact, since just rerendering the whole page because one value update is costly and inefficient, React is very smart about updating only those parts of the page that need to be updated, which is completely hidden from us as developers and this is one of the core and most beautiful features of React.

So, in lines *24-25* we create a *Breadcrumbs* component and pass it *this.state.path* as the *path* property which will be accessible in a *Breadcrumbs* instance via *this.props.path*.

React components can use special XML-like JavaScript syntax extension for defining what they are rendered to. This extension is called **JSX** http://facebook.github.io/jsx/ and we just saw what it looks like in lines *22-29* and *38* where we defined a couple of JSX templates.

What about event handling and interactivity? In line *25* we define the *onChange* attribute which is part of the *Breadcrumbs* component public API, whenever the path will change, the event handler *onPathChange* from lines *10-14* will be called. In line *28* we define the *onClick* handler as *this.reset* so that when the button is clicked we reset the component state in lines *15-19*.

You may remember that for quite a while using inline event handlers and having attributes like *onClick* is considered a bad practice when creating HTML markup. But here it is different since what we are defining in a JSX template is not HTML although it looks like it. Internally, React will create a tree of components, so called **virtual DOM** and will take care of event handling optimization, use event delegation when needed, and the generated HTML will not contain any inline event handlers. So in the context of React inline event handlers are just the way to go. And the *div* from line *27* is not actually an HTML element, but a template based on which a React *div* component will be created in *virtual DOM*, and this component will then be rendered as an HTML *div* in *real DOM*.

Note, how we have two parallel structures that coexist in a React application: *real DOM* which is what we are used to and *virtual DOM*, which is a tree of React components. React takes care of rendering the virtual DOM into real DOM and making sure those are in sync with each other.

Going back to Web Components, let's recall the *shadow DOM* concept. For Web Components the component tree is in *real DOM* and implementation details of components such as *video* are hidden in the *shadow DOM* pieces attached to *real DOM*. For React *real DOM* plays the role of *shadow DOM* and the component tree, *virtual DOM*, is just a tree of React components stored in memory. So here we can clearly see certain parallels between Web Components and React.

#### **Breadcrumbs example**

Now that we have a better understanding of inner workings of React.js apps let's revisit our Breadcrumbs example ported to React https://github.com/antivanov/ui-components/blob/master/React.js/breadcrumbs/breadcrumbs.js

First thing we see is how our component is now composed from other components: *Crumb* and *CrumbSeparator*. For example we can have a structure like this:

* Breadcrumbs
  * Crumb
  * CrumbSeparator
  * Crumb
  * CrumbSeparator
  * Crumb

In other words, *Breadcrumbs* is just a sequence of interleaving *Crumb* and *CrumbSeparator* components which is what it, in fact, looks like on the screen.

```javascript
var Crumb = React.createClass({
  activate: function() {
    this.props.onSelected(this.props.idx);
  },
  onKeyPress: function(event) {
    if (event.nativeEvent.which == 13) {
      this.activate();
    }
  },
  render: function() {
    return (
      <span className="crumb" tabIndex="0" onKeyPress={this.onKeyPress}
        onClick={this.activate}>{this.props.value}</span>
    )
  }
});

var CrumbSeparator = React.createClass({
  render: function() {
    return (
      <span className="crumb-separator"
        title={this.props.tooltip}>{this.props.value}</span>
    )
  }
});

var Breadcrumbs = React.createClass({
  onSelected: function(idx) {
    if (idx < 0) {
      return;
    }
    var newPath = this.props.path.slice(0, idx + 1);

    if (newPath.join('/') != this.props.path.join('/')) {
      this.props.onChange(newPath);
    }
  },
  render: function() {
    var self = this;
    var path = this.props.path;
    var maxEntries = this.props.maxEntries || -1;
    var hasShortened = false;
    var crumbs = [];

    path.forEach(function(pathPart, idx) {

      //Skip path entries in the middle
      if ((maxEntries >= 1) && (idx >= maxEntries - 1) 
        && (idx < path.length - 1)) {

        //Render the dots separator once
        if (!hasShortened) {
          var tooltipParts = path.slice(maxEntries - 1);

          tooltipParts.pop();
          crumbs.push(
            <CrumbSeparator value="..." key={idx}
              tooltip={tooltipParts.join(' > ')}/>,
            <CrumbSeparator value="&gt;" key={path.length + idx}/>
          );
          hasShortened = true;
        }
        return;
      }
      crumbs.push(
        <Crumb idx={idx} value={pathPart} key={idx}
          onSelected={self.onSelected}/>
      );
      if (idx != path.length - 1) {
        crumbs.push(
          <CrumbSeparator value="&gt;" key={path.length + idx}/>
        );
      }
    });

    return (
      <div className="breadcrumbs">
        {crumbs}
      </div>
    );
  }
});
```

*CrumbSeparator* component in lines *18-25* is quite simple, it does not have any interactivity and just renders into a *span* with a supplied tooltip and text content.

*Crumb* in lines *1-16* has some handling for key presses and clicks. In lines *2-4* we call the function *this.props.onSelected* with the index of the current crumb *this.props.idx*, both properties have been supplied by the parent *Breadcrumbs* components in the lines *66-67*.

The *Crumb* and *CrumbSeparator* components are quite self-contained and simple, yet they hide some of the low-level details from the *Breadcrumbs* implementation.

In line *67* we specify that *onSelected* function from lines *28-37* should be called whenever a *Crumb* child component is selected. In *onSelected* we just check if the path has changed compared to what *Breadcrumbs* received in its properties in line *34*, and then call the supplied *onChange* handler in line *35*, as you might remember we defined this handler as a property on the current *Breadcrumbs* instance earlier in *BreadcrumbsDemo*.

Then the implementation of the *render* function in lines *38-80* repeats the logic we already saw in earlier examples for Polymer and Web Components. Note how we can construct the JSX template dynamically in lines *66-67*, *70-71* and inline parts of it later in line *78*. Lines *48-64* deal with the case when the Breadcrumbs component has a specified *maxEntries* property.

Note, how the example is simpler and cleaner than what we did before even with Polymer. This is because quite a many low-level details such as rendering templates and binding data to the views are done for us by React which seems to be a nice bonus to componentization we get.

#### **Key points, philosophy behind React.js**

Let's outline some of the keypoints that the example above demonstrates, we will re-iterate some of them a bit later when we compare different approaches with each other.

React.js is a view only library that solves the challenge of binding data to view and creating reusable component. Data is transformed by React into a component tree and then this tree is transformed into HTML.

Every React component can be viewed as a **function** that takes some arguments via its properties and returns a new value that can include some other React components or React equivalents to HTML elements such as &lt;div&gt;.

Composing your app from React components is quite similar to composing about any program from functions. React encourages a functional style where component definitions are declarative and React handles the low-level details how to make this functional definition of an app work efficiently. Feel free to read more about functional programming http://en.wikipedia.org/wiki/Functional_programming

When looking at React.js one can clearly see certain similarities with a domain specific language for solving a specific problem created in Lisp http://en.wikipedia.org/wiki/Lisp_%28programming_language%29. Just the brackets are a bit different, instead of () it is &lt;&gt; and also React is not general purpose language, but just a view library.

As JavaScript was in part inspired by Scheme (dialect of Lisp) http://en.wikipedia.org/wiki/Scheme_%28programming_language%29 and gives a lot of attention and flexibility to functions, React feels quite natural to use and is following the underlying language philosophy in this regard.
It is very easy to compose and nest functions in JavaScript, and so it is to compose and nest components in React.js. React.js does not try to redefine JavaScript or impose its own vision of the language like Angular.js does to some extent.

But JavaScript is not quite a functional language unlike, for example, Haskell, due to the presence of mutable shared state, so we have to take extra measures when using React to make sure that we do not modify things such as component properties, and more development discipline and effort is required in this respect.

React.js provides a declarative, functional, powerful mechanism of composition and abstraction and is conceptually quite simple because of this.

One minus is that React.js is non-standard, does things in its own way, and the components we create will be reusable only in a React app. Although React is much less intrusive than, say, Angular.js which we will discuss in the next section, and does not dictate how the whole app should be structured. Instead it focuses on a few things: componentization and view updates and solves them.

Another minus is that CSS is still not modular, and unlike in Web Components, styles live completely separately from the JavaScript part of the app.

#### **Flux**

In addition to the view part represented by React, one can also choose to follow the default app arhitecture recommend for React. This architecture is called **Flux** https://facebook.github.io/flux/ and ensures that the data flows to your React components in an organized manner via one control point. React itself just defines how data should be transformed into HTML, and it does not say much about what should be the data lifecycle in the app, Flux tries to complement this gap. But, of course, you can choose any MVC framework you like and use it with React.

#### **Existing component libraries**

You can search for React components here http://react-components.com/ or use React Bootstrap http://react-bootstrap.github.io/components.html or React Material https://github.com/SanderSpies/react-material

# <a name="angular-js"></a>Angular.js

TODO

Plus: everything is included.

Minus: tries to redefine JavaScript, make it look more like Java.
Minus: cascading updates, which scope is which.

# Change of development mental model

A lot like functional composition.

# Notes

TODO: Composition of several components

TODO: Mention libraries of read components for each framework/library

Problems with reusing components between libraries and frameworks, a lot of effort on creating parallel implementations. Will standartization help?

Components are like HTML classes

Componentization solves only one challenge, still there are others to address. Don't expect React to handle everything in your app.
 
jQuery UI plugins are becoming obsolete, patchy approach, no holistic way of organize components together

Plus of WebComponents: using standard APIs together with your components

Shadow DOM in WebComponents  real generated HTML for React.js and Angular

React and Angular lock you in into using them, React less so

React and Angular handle templating, data binding, etc.

Web Components standards give you complete freedom what libraries or frameworks you may choose to use

Other libraries and frameworks that allow to create components