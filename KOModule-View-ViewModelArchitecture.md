# KO Module-View-ViewModel Architecture #
*By Michael Berkompas ([github/mberkom](https://github.com/mberkom))*


Pioneered by Microsoft in their SilverLight platform, the MVVM architecture empowers UI heavy web apps to write their client-side JavaScript cleanly and without unnecessary glue between the model and UI.  [KnockoutJS](http://knockoutjs.com/), the leading JavaScript implementation of the MVVM pattern, has become a popular library for writing MVVM UIs for the browser.  While certainly powerful, vanilla KnockoutJS leaves much to be desired when writing large multi-page web applications.  This article provides some insights into how you can build upon the featureset of KnockoutJS and write scalable UIs for large web applications.


First, let’s define the requirements of a large-scale web application.  


1. The code needs to be organized.  Finding files and knowing what they contain should be intuitive.
2. The code should be divided into individual stand-alone modules.  Reusability is very important in a large web application.  You can solve this problem by dividing your code up into individual components that then declare their own dependencies.
3. Because web UIs often demand many different interactions for one page, each individual module of code must be able to be added and removed dynamically.  
4. There should be a simple way to load data from a data-source into and out of the UI.
5. The code should be easy to debug.


Now that we’ve got that out of the way, let’s see how vanilla KnockoutJS lines up.


1. (Organized Code) - KnockoutJS doesn’t provide much in the way of code organization.  In reality, it’s a library, and not a true framework.  However, there are best practices about how to initialize your viewmodels and where you should put your ko.applyBindings() call.  For a large-scale web application, we need more than this.
2. (Stand-alone Code Modules) - KnockoutJS doesn’t understand the concept of a code module.  However, it does work well with [RequireJS](http://requirejs.org/) and AMD.  
3. (Start & Stop Modules) - Knockout allows you to fire up multiple instances on one browser page, but requires significant boilerplate code to remove an instance completely.  This can be mitigated by writing an object that provides start & stop methods that your KO modules can then inherit from. I call this object the “Module” and it’s explained more later.
4. (I/O with data from a server) - Knockout viewmodels can contain methods for parsing out a clean model as well as serializing a data feed into a KO viewmodel.  KO comes built-in with AJAX methods, but [jQuery](http://jquery.com/) is probably a better idea to create those interactions.
5. (Debuggable code) - Unfortunately, due to the fact that KO declares its bindings in HTML, many binding related errors are difficult to debug.  It becomes especially difficult to debug when you use a service like [TraceKit](https://github.com/csnover/TraceKit) to log JS errors from your users.  For this reason and several others, the default view-viewmodel binding mechanism in Knockout will need to be replaced in order to support scalable web applications.  


Okay, so Knockout needs to be extended a little.  In fact, the generic MVVM pattern doesn’t really work when you have a bunch of interactions, modules, pages, etc...  The answer is to replace it with something very similar.  


Enter the Module-View-ViewModel(s) architecture.  You can think of the “Module” portion as a self contained widget that declares its own Knockout instance.  The “View” stays basically the same and remains HTML, but the way bindings are declared and accessed changes. The “ViewModel(s)” are a set of viewmodels specific to the module.  They each contain a serialize method that returns a clean model and a parse method that accepts a clean model.  


**1. Module**


The module is a JS object that provides methods for starting and stoping a KO instance.  Included are properties for an HTML template, and declaring your bindings.  It also contains the parent module viewmodel which controls all the KO interactions.  Here’s an example of what it could look like.

```javascript
/**
Prototype object for creating individual KnockoutJS modules

@module Shared
@class ko-module
@namespace
@static
**/
define(["knockout", 'jquery', 'Shared/js/helpers'], function (ko, $, helpers) {
    var module = {};
    /**
    KO viewModel
    @property vm
    @static
    **/
    module.vm = {};
    /**
    KO bindings
    @property bindings
    @static
    **/
    module.bindings = {};
    /**
    Optional binding namespace to keep bindings from being overwritten by another module
    @property bindingNamespace
    @static
    **/
    module.bindingNamespace = null;
    /**
    Whether bindings are registered with the KO classBindingProvider or not
    @property bindingsRegistered
    @static
    **/
    module.bindingsRegistered = false;
    /**
    The HTML node wrapping the area we wish to apply KO bindings
    @property $wrapperNode
    @static
    **/
    module.$wrapperNode = $("body");
    /**
    Optional html string to be loaded in as a template into this.$wrapperNode
    @property template
    @static
    **/
    module.template = null;
    /**
    Injects html, registers and applies bindings, calls viewmodel `init` method
    @method start
    @static
    **/
    module.start = function ($wrapperNode, initArgs) {
        // Update wrapper node
        if ($wrapperNode) this.$wrapperNode = $wrapperNode;

        this._openNodeId = "mod_" + helpers.uniqueId();
        var $targetNode = $wrapperNode;

        // Insert template html
        // Wrap in a div
        if (this.template) {
            this.$wrapperNode.html($("<div></div>").attr("id", this._openNodeId).html(this.template));
            $targetNode = $wrapperNode.find("#" + this._openNodeId);
        }

        // Register bindings
        // Class binding provider has to be set up first...
        if (!this.bindingsRegistered) {
            this.bindingsRegistered = true;
            var register = this.bindings;
            if (this.bindingNamespace !== null) {
                register = {};
                register[this.bindingNamespace] = this.bindings;
            }
            ko.bindingProvider.instance.registerBindings(register);
        }

        ko.applyBindings(this.vm, $targetNode[0]);

        // If VM has an Init method, call it with any initArgs that may have been passed.
        if (this.vm.init && _.isFunction(this.vm.init)) {
            this.vm.init.apply(this, (initArgs || []));
        }
    };
    /**
    Removes html, and un-applies bindings
    @method stop
    @static
    **/
    module.stop = function () {
        var $openNode = $("#" + this._openNodeId);	

        // Unbind event handlers
        $openNode.find("*").each(function () {
            $(this).unbind();
        });

        // Remove KO subscriptions and references
        if (this.template !== null) {
            ko.removeNode($openNode[0]);
        } else {
            ko.cleanNode($openNode[0]);
        }
    };
    module.end = module.stop;


    return module;
});
```

**2. The View**


Views continue to be HTML, but instead of using the default KO method of declaring a data-bind attribute that contains our bindings, we declare our bindings in our module and reference them via a key very much like css.  To do this, we use the [Class Binding Provider](https://github.com/rniemeyer/knockout-classBindingProvider), a project started by Ryan Niemeyer.  The bindings are declared like this: 

```javascript
var bindings = {
   title: function(context, classes) {
       return {
           value: this.title,
           enable: context.$parent.editable
       }
   },
   input: {
       valueUpdate: 'afterkeydown'
   },
   list: {
       items: function(context, classes) {
           return {
               foreach: this.items
           }
       }
   }
};
```

And referenced like this: 

```html
<ul data-class="list.items">
   <li> ... </li>
</ul>
```

There are several benefits to declaring the bindings in JS.  One of the most important is that your module can take care of registering them and can also maintain control over the bindings.


1. The markup can stay clean and simple
2. Bindings can be reused, even at different scopes
3. You can set breakpoints in the bindings to inspect the data being passed through them making debugging comparatively simple.
4. You can do logging in the bindings to understanding how many times they are being called
5. You can change/alter the bindings on an element whenever your bindings are triggered
6. Bindings go through less parsing (do not need to go from a object literal in a string to code)

See an example implementation in the TodoMVC labs [here](http://todomvc.com/labs/architecture-examples/knockoutjs_classBindingProvider/).


**3. ViewModel**


Viewmodels stay basically the same.  However, because there is no longer a concept of models, your viewmodels will all need to provide methods for serializing and parsing out and into clean data models.  Here’s an example:

```javascript
var volumeDiscount = function (setupData) {
    var self = {};
    setupData = setupData || {};


    self.quantity = ko.observable(setupData.quantity || 0).extend({ numeric: 0 });
    self.markup = ko.observable(setupData.markup || "");


    self.serialize = function() {
        return {
            quantity: self.quantity(),
            markup: self.markup()
        };
    };


    return self;
};
```

## Conclusion ##


Combined with a couple other libraries and the Module-View-ViewModel architecture, KnockoutJS can truly become an excellent solution for large-scale web applications.  Let’s go back over the five requirements for a large-scale web app and see how this slight variation on the MVVM architecture works out.


1. (Organized Code) - We can split our code apart into files and folders that are named according to their contents.  Because we understand things as Modules, we expect that there would be a primary module file, a viewmodel(s) file, and at least one HTML view. 
2. (Stand-alone Code Modules) - By using RequireJS to load up our modules and handle dependencies, we can now write our code in an organized, modular, and testable way.  
3. (Start and Stop Modules) - The concept of a module provides us with the boilerplate code to start and stop a KO instance as well as handle dynamically loaded html.  
4. (I/O with data from a server) - Viewmodels provide a clean way to get at their data, as well as initialize themselves with setup data.  
5. (Debuggable code) - Because we declare bindings in the JS module and reference them using the Class Binding Provider, all our code becomes comparatively easy to debug. 


With that overview of the possibilities for large-scale web applications, I hope you’ll consider using it in your next project.  MVVM can be a scalable and competitive solution for advanced web application interfaces.