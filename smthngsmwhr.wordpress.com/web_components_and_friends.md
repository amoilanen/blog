# Web App Development Challenges

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

# Frameworks, libraries, standards... 

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

# Need of Componentization

You might have heard of domain-specific languages (DSLs) http://martinfowler.com/books/dsl.html and such before in relation to other areas of software development and not necessarily front-end or Web development. Feel free to read more on the subject in case you are interested, although this is not required to understand the present article. In brief the main idea is that in order to solve some challenging problem it often pays off to first create a language in which this problem is then easily solvable. Then having this language will allow you to describe the system and its behavior much easier. One good example would be SQL to manipulate and query data stored in a relational database. Try doing that in your plain code and you will be a world of pain and maintainability nightmare if your data schema and queries are complex enough.

The idea is that as Web applications become more complex, they are in many respects not that different from other areas of Software Development and can certainly benefit from creating DSLs. In fact HTML is already such a DSL albeit quite a low level and general purpose one. HTML5 introduces some new semantic tags such as &lt;section&gt;, &lt;header&gt;, &lt;footer&gt;, etc. but this still is not high enough level of abstraction if you develop a large app. What we would like to have is to operate with more high things, in the case of a Web store this might be: &lt;cart&gt;, &lt;searchbox&gt;, &lt;recommendation&gt;, &lt;price&gt;, etc.

Of course if your app is quite simple and is just a couple of pages you may not need so much componentization and code reuse. Compare this to the case when you need to write some simple script in, say, Python, then you may not need to define any classes or even functions in that script. Still, it is nice to have the ability to create reusable abstractions (functions, classes) when there is a genuine need for them. Components in a Web app are not all that different from functions or classes in this regard.

The obvious benefits of creating and using components are better maintainability, code reuse and speed of development once you already have the necessary components developed. Components are like building blocks from which you can build your app, if those blocks are well-designed you can just quickly throw together some functionality that before would require a lot of boiler-plate coding. Same as with functions and classes, but just on a bit different level.

Hopefully, by now you are convinced in the potential usefulness of components and would like to know more about the technical details. So far our discussion has been a bit abstract and philosophical, it is right about time to delve into examples and see how these ideas are used in practice.

# Examples

We chose to implement a simple Breadcrumbs component using different technologies listed above on this page with examples http://antivanov.github.io/ui-components

The source code is available here https://github.com/antivanov/ui-components, feel free to check out and modify it, and we will go deeper into various frameworks and libraries in the next sections, as well as will cover some of the code in the examples.

# Web Components

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

# Notes

TODO: Composition of several components

Problems with reusing components between libraries and frameworks, a lot of effort on creating parallel implementations. Will standartization help?

Components are like HTML classes

Componentization solves only one challenge, still there are others to address. Don't expect React to handle everything in your app.
 
jQuery UI plugins are becoming obsolete, patchy approach, no holistic way of organize components together

Plus of WebComponents: using standard APIs together with your components

Shadow DOM in WebComponents  real generated HTML for React.js and Angular

React and Angular lock you in into using them, React less so

Web Components standards give you complete freedom what libraries or frameworks you may choose to use