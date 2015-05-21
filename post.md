#1. Introduction
AngularJS, is currently one of the most widely used client side framework. It exposes large amounts of complex functionality, and lends itself very well to writing reusable, generic code.
This article will attempt to be a detailed, and heavily opinionated look at what is Angular's Unit of reuse - directives, with an eye towards explaining their many frequently confusing features. 

###1.1 Requirements, Assumptions
This article is going to assume some familiarity with the JavaScript programming language and AngularJS. To really get the most out of this article, it would be best if you had already attempted to do some work in AngularJS, but had yet to really attempt to modularize and reuse, or dive deep into directives. The details of the language outside of when it is directly applicable to the subject at hand (e.g. the digest lifecycle, dependency injection), will NOT be expounded upon.

#2. Directives at 10,000 feet
At their core, directives are functions which runs when a DOM element they have been attached to is encountered in the DOM tree.

###2.1 Anatomy of a directive, at 10,000 feet

Lets start, with a directives type signature:

    injectablesList -> configObject
    
and proceed on to an annotated, non functional example:

    angular.module('annotated').directive('demoOne', ['injectable', function (injectable) { //the directive definition function, follows standard dependency injection rules
    //Code can be run here before returning
      return { //must return an object. Technically its entire contents is optional
        restrict: 'EACM', //the DOM type of the directive
        scope: { //isolate scope parameters
          item: '@',
          item2: '=',
          item3: '&'
        },
        transclude: 'true', //transclusionType
        controller: function ($scope) { //controller function}, 
        require: 'foo', //require other controllers here
        template: '<div> I am awesome!</div>,
        compile: function (elem, attr, transcludeFn) { //compile function
          return { //must return an object
            pre: function (scope, elem, attr, controllers, transcludeFn) { //pre link function },
            post: function (scope, elem, attr, controllers, transcludeFn) { //post link function }
          }
        },
        controllerAs: 'name' //controller name for DOM interpolation,
        priority: '1000' //compilation priority,
        terminal: false // is this the last compileable directive
        }
      }]);
      
Well, now we've cleared that up, our work here is done.
O, right.....

#3.Directives at the MicroScopic level
The above directive contains three fundamental building blocks. The first, is the directive signature. This is the annotated function and injectable definition attached to a module. In the code sample above:

    angular.module('annotated').directive('demo-one', ['injectable', function (injectable) { //the directive definition function, follows standard dependency injection rules
    
The directive signature follows Angular's standard dependency injection syntax and rules.
The second is the directive function's body prior to the return statement. This is a convenient place to define functions reused within different inner functions of your directive, or to configure things for use.

The third and final part of the directive is the directives mandatory returned object. This object instructs the framework on how to construct a directive when the snake-cased version of the directives name (`demo-one` in our current case) is encountered in the DOM.

###3.1 The Directive Definition Object
This is where the meat of the directive's functionality lives, and where we will be spending the vast majority of our time.
#####3.1.1 restrict
The first and simplest configuration parameter available on the returnable object is the `restrict` key. It accepts one (or multiple) of the letters `EACM`, representing `Element, Attribute, Class`, and `Meta`. This is the specific type of element marker that AngularJS will accept for this directive. Specifically, given our sample directive signature 
    angular.module('sample').directive('demoOne', function () {
      return {
        restrict: 'E'
      }
    }
    
`restrict: 'E'` means that the directive will trigger in the following circumstance: `<demo-one/>`
`restrict: 'A'` means that the directive will trigger in the following circumstance: `<div demo-one/>`  
`restrict: 'C'`  means the directive will trigger in the following circumstance:
`<div class='demo-one'/>`  
`restrict: 'M'` means the directive will trigger in the following circumstance:
`<!--directive: demo-one-->`

These can also be combined.
In general I dislike using class restricted directives as it needlessly couples functional directive information with css presentation logic, and makes the stylesheets significantly more brittle.
I also similiarly dislike meta restricted directives as they pollute the codebase with needless comments.
In general attribute level directives are more flexible then element level directives as they can be composed with ease simply by putting multiple directives on the same element.

#####3.1.2 Templating
There are two keys that can be passed into a Directive Definition Object to indicate its HTML structure. The first is the `template` key which accepts an HTML string such as `<div> I am in your Directive Definition Object rendering your content</div>` directly. The second is a `templateUrl` key which accepts a path to an html file. By default these will override any DOM originally nested within the directive. So:

    <my-awesome-directive><div> Content was here, but now it ain't</div></my-awesome-directive>

Will override the inner `div` with whatever is defined on the `template` or `templateUrl` of `<my-aweosme-directive>`. If you would like to see how that can be overridden, scroll to section 3.1.4.

#####3.1.3 Scope
The scope parameter of the Directive Definition Object can take three values:
1) `scope: false`. This is the default value and instructs your directive to share its scope with the parent scope. Any changes done on this scope will automatically be done to the parent also.
2) `scope: true`. This instructs the directive to create a new child scope which inherits from the parent prototypically. An in depth discussion of what prototypical inheritance means specifically is out of the scope of this article, the salient point is this:
Given the following DOM:
    
    <body ng-app="demo">
      <div ng-controller="outerCtrl">
        <div>{{outerVal}}</div>
        <div>{{model.innerVal}}</div>
        <div>{{innerModel.innerVal}}</div>
        <inner-dir></inner-dir>
      </div>
    </body>
and the following javascript:
  
    angular.module('demo', []).controller('outerCtrl', function ($scope) {
      $scope.outerVal = "outer";
      $scope.model = {innerVal: "inner"};
    })
    .directive('innerDir', function () {
      return {
        restrict: 'E',
        scope: true,
        link: function (scope) {
          scope.outerVal = "inner";
          scope.model.innerVal = "inner2";
          scope.innerModel = {innerVal: "innerVal"};
        }
      }
    });
The printed result will be `outer` and `inner2`, and nothing. Said a more generic way, references will be shared from the outer scope to the inner scope, but not from the inner scope outwards.
A running demonstration can be found here:
http://plnkr.co/edit/7OrA3Sw6uxOixkF22UOA

A quick aside: Astute readers may notice the `link` key used without further discussion. In brief, it permits for the running of arbitrary javascript by the directive when it is loaded on a DOM element. For a deeper discussion, scroll to section 3.1.5.

3) `scope:{}`: This is known as isolateScope, and is worthy of its own discussion.
#####3.1.3.1 Isolate Scope
Isolate Scope is a way to pass individual things from the parent scope into the directive scope, without inheriting everything. There are three methodologies for passing scope properties. The first is attribute binding also known as one way binding, and is done with an `@` sign:

    .directive('attributeBound', function () {
      return {
        restrict: 'E',
        scope: {
          oneWay: '@'
        },
        template: '<div>{{oneWay}}</div>',
        link: function (scope) {
          scope.oneWay = 'newValue';
        }
      }
    }
then calling this directive like so:
    <attribute-bound one-way='{{outerValue}}'/>
Will bind a variable `scope.oneWay` in the directive to the value of `scope.$parent.outerValue`, at directive compilation time and will NOT propagate changes to the value out onto the outer scope. The important thing to realize with one way bindings, is that they will *always* occur as a string. What that means in practice is that if you would like to pass the contents of a variable you *must* interpolate it as demonstrated above, since merely setting it to `one-way='outerValue'` would pass the literal string, `outerValue`. The second is that if the contents of the interpolated variable is an object or an array, you must apply `JSON.parse` in your directive, like so:

    scope.oneWay = JSON.parse(scope.oneWay);

The second method of passing variables onto an isolate scope is commonly referred to as either reference or two-way binding. 

    .directive('referenceBound', function () {
      return {
        restrict: 'E',
        scope: {
          twoWay: '@'
        },
        template: '<div>{{oneWay}}</div>',
        link: function (scope) {
          scope.twoWay.value = 'newValue';
        }
      }
    }
    
which can then be invoked with 

    <reference-bound two-way="outerValue"/>
There are several things to note here. The first is that unlike in the attribute example, interpolation is unnecessary as object references are passed directly. The second is that any changes to `scope.twoWay` in the directive will propagate to `scope.$parent.outerValue` (the outer scope), while any other changes to the directive scope not also listed on the isolate scope will not.  

The third and final method of passing information into a directive's isolate scope is known as expression binding or (in my head) block binding (my Ruby is shining through and I can't resist. Also that analogy helped me grasp it to begin with). It is represented by the `&` symbol and is used to pass function references in the parent scope.

    .directive('expressionBound', function () {
      return {
        restrict: 'E',
        scope: {
          innerValue: '@',
          outerFunction: '&' 
        }
      }
    }
    
Which can then be invoked with:

    <expression-bound inner-value='awesomeString' outerFunction='awesomeClickHandler' ng-click='outerFunction({outerParamName: innerValue})'/>
    
There are several things worth noting. First and foremost, while this example is (deliberately) trivial, you could absolutely massage innerValue in the directive, or call the outerFunction from the directive javascript directly. The second is that when an expression bound function is called, it must name the params to coincide with the signature of the function to which it is a reference. In this particular case what that means is that in the outer scope, the function signature for `awesomeClickHandler` is:

    function awesomeClickHandler (outerParamName) {
Clear as mud.... right?
#####3.1.3.2 When to use which scope?
The guidelines as I see them are:  
Use `scope: false` if your directive is a wrapper around templates that *does not modify or read shared mutable state*.  Static DOM structures et al.  
Use `scope: true` if you wish your directive  to have access to all of the parent scope but do not wish do modify it. Try to avoid modifying object references on the parent. Thing of this as a closure or a read only scope.  
Use `scope: {}` (isolate scope) for reusable, self contained components. A note: an isolate scope will force all other directives on the same element to use that isolate scope. It is not legal to have multiple directives with isolate scope on the same element.

#####3.1.3.4 A scope example
Let's say we start out wanting to render a team, and then realize we want to render a list of things, such that the definition of item deletion is up to the consumer, but the deletion occurs on a list item basis.

Fully functional source code here: http://plnkr.co/edit/6u7EDTUF2exJusdNQ6PG

A short review:

    <body ng-app="isolateScope">
        <div ng-controller="configController">
        <nested-list type="{{typeOfThing}}" ,="" list="collection" delete-func="deleteFunction(id)"></nested-list>
        </div>
    </body>
    
    var app = angular.module('isolateScope', []);

    app.service('mockDeletionService', function () {
      this.delete = function (id) {
        //$http call to actually delete goes here
      }
    })

    app.controller('configController', function ($scope, mockDeletionService) {
      $scope.collection = [{id: 1, name: 'Jim'}, {id: 2, name: 'John'}];
      $scope.typeOfThing = "Team Member";
      $scope.deleteFunction = function (id) {
        console.log(id);
        mockDeletionService.delete(id);
        for (var count = 0; count < $scope.collection.length; count++) {
          if ($scope.collection[count].id === id) {
            $scope.collection.splice(count, 1);
            break;
          }
        }
      }
    });
    
    app.directive('nestedList', function () {
      return {
        restrict: 'E',
        scope: {
          deleteFunc: '&',
          type: '@',
          list: '='
        },
        template: '<list-item type="{{type}}" del="innerDel(item.id)" item="item" ng-repeat="item in list">',
        link: function (scope) {
          scope.innerDel = function (id) {
            scope.deleteFunc({id: id});
          }
        }
      }
    });
    
    app.directive('listItem', function () {
      return {
        restrict: 'E',
        scope: {
          del: '&',
          type: '@',
          item: '='
        },
        template: '<div><span>{{type}}: {{item.name}}</span><button ng-click="del()">Delete {{type}}</button></div>'
      }
    });

Things to note:   
The function pointer is passed to a deeper level of nesting. This could repeat an arbitrary number of times.  
Modifying the collection at the outer level modifies the collection at the inner level as it is passed by reference
#####3.1.4 Transclusion
Wikipedia defines transclusion as:  

    In computer science, transclusion is the inclusion of part or all of an electronic document into one or more other documents by reference. 
    http://en.wikipedia.org/wiki/Transclusion
    
This is honestly a pretty good definition. So how does this work in Angular?
As we know from section 3.1.2 any DOM written inside a directive is overridden by default. But let's say we wanted to inject DOM, for example a header that could vary depending on the type of thing we are listing and deleting? I'm glad you asked, lets modify our previous example.
#####3.1.4.1 Transclusion: the base case
http://plnkr.co/edit/9uK11wQ8EGMNVeWyO1Y2

Since we are building on our previous example, I will not paste the whole code here, rather simply highlighting the differences.

First, we have removed `type: '@'` from the list-item isolate scope.    
Second, we have added `transclude: true` onto the Directive Definition Object of the list item.
Thirdly we have changed the templates:

    template: '<list-item del="innerDel(item.id)" item="item" ng-repeat="item in list">' +
      '<span>It\'s a bird, it\'s a plane, it\'s transclusionMan. {{type}}' +
      '</list-item-type>',
      
On the list, and 

     template: '<div><div ng-transclude>The transcluded header goes here: </div><span>{{item.name}}</span><button ng-click="del()">Delete {{type}}</button></div>'
     
on the list item. Note that passing the type into the list item is superfluous as the transcluded DOM executes in a prototypical descended of the parent (in this case a child of the list's isolate scope). Also note that the div labeled with `ng-transclude` has its content wiped out and replaced with the transcluded DOM.

#####3.1.4.2 Transclusion: the complex case
Let's say we decided to write a custom widget. It should accept a header to be displayed from the outside, and a header to be displayed on the inside. How can we achieve this?
The first thing to note is that the `link` function accepts the following parameters:
`scope`, the directives scope    
`elem`, the directives element
`attrs`, the element attributes
`controller`, the required controller, see section 3.1.5
`transcludeFn`, a function accepting the directive scope, and a function accepting the cloned transclude element, and the transclusion scope.
A more detailed discussion will follow in section 3.1.5

    app.controller('configController', function ($scope) {
      $scope.item = {id: 1, name: 'Jim'};
      $scope.typeOfThing = "Awesome Person";
    });
    
    app.directive('headeredThing', function () {
      return {
        restrict: 'E',
        transclude: true,
        scope: {
          item: '='
        },
        template: '<div>{{item.name}}</div>',
        link: function (scope, elem, attrs, controller, transcludeFn) {
          transcludeFn(scope, function(tElem, tScope) {
            for (var count = 0; count < tElem.length; count++) {
              if (tElem[count].attributes) {
                var classes = tElem[count].attributes.getNamedItem('class').value.split(' ');
                if (classes.indexOf('outer-header') !== -1) {
                  angular.element(tElem[count]).insertBefore(elem);
                } else if (classes.indexOf('inner-header') !== -1) {
                  angular.element(tElem[count]).insertBefore(elem.find('div'));
                }
              }
            }
          })
        }
      }
    });
    
    <body ng-app="transclusion">
      <div ng-controller="configController">
        <headered-thing item="item"><div class="outer-header">I am your outer header! This transclusion thing is easy after all!</div><div class="inner-header">It's a bird, it's a plane, it's transclusionMan.</div></nested-list>
      </div>
    </body>
    
Working code: http://plnkr.co/edit/cCynGS7gxL1Q6pvaEd3H

What is going on here you may ask? In short, we are using our transclusion function to rip apart the transcluded DOM, and insert it into our HTML as appropriate. All of the DOM level manipulation is off the native `NamedNodeMap` object as defined here:
https://developer.mozilla.org/en-US/docs/Web/API/NamedNodeMap

#####3.1.4.3 Transclusion: the (slightly)even more complex case
What if we wanted to extend the above example to also print a value from the parent scope in our transcluded headers? 

modifying the first parameter of the `transcludeFn` from `scope` to `scope.$parent` does the trick. In general the transcluded content can be transcluded from absolutely any scope as long as you can find a reference to it.

http://plnkr.co/edit/cCynGS7gxL1Q6pvaEd3H

#####3.1.5 Directive Compilation
The Directive Definition Object can have the following keys:    
`compile`. A compile function useful to transform the DOM template. Returns an object containing `pre:` and `post:`. Both represent linking phases.
`link`. This is identical to the `post` returned from the `compile` function. This and `pre` can be loosely thought of as the logical workhorses, where scope based logic lives.
`priority`. The order in which directives will execute.
`terminal` Whether a directives compilation should terminate further compilation on the current page. Will also prevent any child directives from compiling.
`controller`. This is a good place to expose API's for consumption by other directives.

The important thing to remember is that compilation happens first, from the outside in, and in priority order, followed by the controller function, from the outside in, in priority order, followed by the preLink function, from the outside in, in priority order, followed by the postLink function from the inside out, in reverse priority order. 

Example:
	app.directive('outerCompile', function () {
	  return {
	    restrict: 'E',
	    template: '<div inner-compile-first inner-compile-second inner-never-compile/>',
	    controller: function () {
	      console.log('outer controller');
	    },
	    compile: function () {
	      console.log('outer compile');
	      return {
	        pre: function () {
	          console.log('outer prelink');
	        },
	        post: function () {
	          console.log('outer postlink');
	        }
	      }
	    }
	  }
	});
	
	app.directive('innerCompileFirst', function () {
	  return {
	    restrict: 'A',
	    priority: 10000,
	    template: '<inner-nested-compile/>',
	    controller: function () {
	      console.log('inner first controller');
	    },
	    compile: function () {
	      console.log('inner first compile');
	      return {
	        pre: function () {
	          console.log('inner first prelink');
	        },
	        post: function () {
	          console.log('inner first postlink');
	        }
	      }
	    }
	  }
	});
	
	app.directive('innerCompileSecond', function () {
	  return {
	    restrict: 'A',
	    priority: 1000,
	    terminal: true,
	    controller: function () {
	      console.log('inner second controller');
	    },
	    compile: function () {
	      console.log('inner second compile');
	      return {
	        pre: function () {
	          console.log('inner second prelink');
	        },
	        post: function () {
	          console.log('inner second postlink');
	        }
	      }
	    }
	  }
	});
	
	app.directive('innerNeverCompile', function () {
	  return {
	    restrict: 'A',
	    priority: 10,
	    controller: function () {
	      console.log('never controller');
	    },
	    compile: function () {
	      console.log('never compile');
	      return {
	        pre: function () {
	          console.log('never prelink');
	        },
	        post: function () {
	          console.log('second to never postlink');
	        }
	      }
	    }
	  }
	});
	
	app.directive('innerNestedCompile', function () {
	  return {
	    restrict: 'E',
	    controller: function() {
	      console.log('inner nested controller');
	    },
	    compile: function () {
	      console.log('inner nested compile');
	      return {
	        pre: function () {
	          console.log('inner nested prelink');
	        },
	        post: function () {
	          console.log('inner nested postlink');
	        }
	      }
	    }
	  }
	});
	
With the HTML:
    
    <outer-compile></outer-compile>

The console then reads:

	outer compile
	inner first compile
	inner second compile
	outer controller
	outer prelink
	inner first controller
	inner second controller
	inner first prelink
	inner second prelink
	inner second postlink
	inner first postlink
	outer postlink

So there are several things to note. The first is that even though the terminal was on the inner second directive, the template from the inner first directive did not compile. The second is that the function ordering takes precedence over the directive order. Specifically, 
The compile functions run as a block before the rest of the functions, however the link and controller functions run in priority order. 

Code: http://plnkr.co/edit/FgJM0SMraIP2dmorvHTx

Lets dive deeper into these different functions and figure out what to use them for.

#####3.1.5.2 Compilation
The `compile` function of a directive does not have access to scope. It accepts the following parameters:
`tElem`, the template Element
`tAttrs`, the template Attributes
It runs before the template Element is rendered. What that means in practice, is that any change done to either `tElem` or `tAttrs` will propagate to all instances of this directive. The flipside that this is a great place to do that kind of manipulation as it will only ever run once unless explicitly recompiled.

#####3.1.5.3 Controller
This will run before the isolateScope binds and before any nested DOM is linked. This is a great place to do scope initialization and massage, as well as providing an opportunity to expose an API to other directives. What that means is that you can expose functions and properties off the object returned by the controller, which can then be pulled into other directives with the `require` key. For a more in depth discussion with examples of the latter, please see 3.1.6.
One gotcha worth mentioning is that this follows standard Angular Dependency Injection Syntax, so things like:

    controller: function ($scope) {...}

will break at minification. As usual, the fully qualified syntax:

    controller: ['$scope', function ($scope) {}]
is preferable.

#####3.1.5.4 Linking
There are two linking functions both of which have the signature
    
    function(elem, attrs, scope, controller, transcludeFn)

#####3.1.5.4.1 PreLink
This will run after the isolateScope binds but before any nested DOM is linked. In general, I prefer to do my scope instantiation here rather then in the controller for semantic reasons. Specifically so as to permit Controllers to serve exclusively for the purposes of compositionality through API surface exposure as discussed in 3.1.6. It is very important NOT to query nested DOM.

#####3.1.5.4.2 PostLink
This is the workhorse and default function called by a directive. If no compile function is provided and only a `link` key is a given, it defaults to this. It will run after all nested directives have fully compiled and linked as well as any directives with a higher priority.

#####3.1.6 Require
The final key is `require`. Fundamentally what this allows you to do is to pull in one (or multiple) directive controllers to expose them as an API. 
The controller name can be prefixed with `^` to indicate that the directive exposing the controller must be above the current element in the DOM, `?`, to indicate that the controller is optional, or `^?` to indicate it is up AND optional. Lack of a prefix indicates that the directive must be a sibling. For the latter reason (the possibility to use an API exposing directive as a sibling), it is best practice NOT to use Element level directives for this purpose.
The controller (or array of controllers if multiple) is then passed into the linking functions as the fourth parameter. Recall:
   
    link: function (scope, elem, attrs, controller, transcludeFn)    
    
This opens the field to multiple interesting strategies.

Strategy 1) Bundle functionality into an API level directive. Commonly used tasks like sorting and filtering lend themselves well to this kind of implementation. For example lets say we want to build a directive that can sort an arbitrary collection.

We could then write our directive:

	app.directive('sortable', function () {
	  return {
	    restrict: 'A',
	    controller: function () {
	      this.sort = function (list, comparator) {
	        list = list.sort(comparator);
	      }
	      return this;
	    }
	  }
	});

And consume it like so:

	app.directive('demoList', function () {
	  return {
	    restrict: 'E',
	    require: '^?sortable',
	    template: '<div ng-repeat="item in list">{{item.name}}</div>',
	    link: function (scope, attrs,elem, ctrl) {
	      scope.list = [{name: 'John'}, {name: 'Alan'}, {name: 'Cindy'}, {name: 'Scarlett'}, {name: 'Aristotle'}, {name: 'Barack'}];
	      scope.sort = function () {
	        ctrl.sort(scope.list, function (p1, p2){
	          if(p1.name < p2.name) {
	            return -1;
	          } else if (p1.name === p2.name) {
	            return 0;
	          } else {
	            return 1;
	          }
	        });
	      };
	    }
	  }
	});

And lo, they saw that the sort was in place, and that it was flexible with respect to its application and they saw that it was good.

Working example: http://plnkr.co/edit/LuD2jBdGfmw1NhUPMZPh

The other application of directives with `require` allow reaching across isolate scope boundaries with ease. This is due to the fact that Javascript will scope variables to their least permissive point. What that means is that if a controller function references `$scope`, and then a directive requires it and calls the function, it will be operating on `$scope`.

For example:

	app.directive('foreignScope', function () {
	  return {
	    restrict: 'A',
	    controller: function ($scope) {
	      $scope.list = [{name: 'John'}, {name: 'Alan'}, {name: 'Cindy'}, {name: 'Scarlett'}, {name: 'Aristotle'}, {name: 'Barack'}];
	
	      this.add = function () {
	        $scope.list.push({name: "new Guy"});
	      }
	      return this;
	    }
	  }
	});
	
	app.directive('demoList', function () {
	  return {
	    restrict: 'E',
	    template: '<div ng-repeat="item in list">{{item.name}}</div>',
	    link: function (scope, attrs,elem, ctrl) {
	      scope.list = [{name: 'John'}, {name: 'Alan'}, {name: 'Cindy'}, {name: 'Scarlett'}, {name: 'Aristotle'}, {name: 'Barack'}];
	    }
	  }
	});
	
	app.directive('demoIsolate', function () {
	  return {
	    restrict: 'E',
	    template: '<nested-isolate/>',
	    scope: {}
	  }
	});
	
	app.directive('nestedIsolate', function () {
	  return {
	    restrict: 'E',
	    scope: {},
	    require: '^?foreignScope',
	    template: '<div ng-click="callAdd()"> CLICK ME!</div>',
	    link: function (scope, elem, attrs, controller) {
	      scope.callAdd = function () {
	        console.log('foo');
	        controller.add();
	      };
	    }
	  }
	});

Note the nested isolate Scopes. Now given the following DOM:

	<div foreign-scope>
	   <demo-list list='list'></demo-list>
	   <demo-isolate/>
	</div>
A click on the giant CLICK ME! will in fact add the new guy! Obviously this is a slightly fanciful example, but with a little bit of imagination the utility should become rather obvious as a replacement for deep passes of `&` isolate scope bindings.
Working Example: http://plnkr.co/edit/xGozv2RHZz7CH0jFuyvA

#4 Other Composition Strategies
So what other composition strategies are there? We've talked about isolate scope as a tool for self sustainability and reuse, and we've talked about controllers and require as tools of composition.

There are several patterns worth mentioning. 

First and foremost, by injecting `$inject`, you can dynamically identify different services depending on the situation. This is a pattern that can be quite easily used with an `@` binding. All it requires is that the varying injectable services have a uniform API. This allows building flexible components that inject different type of things in different situations. This pattern is complimentary with everything discussed so far and can be used quite easily in conjunction with the controller/require pattern and isolate scopes. Think of controllers/require as exposing functionality, while `$inject` and services expose data.

Another common strategy is to use the `attrs` parameter into the linking function, in conjunction with `attrs.$observe` to communicate among sibling directives. The important thing to remember here, is that the attrs object is shared between sibling directives by reference, changes in one place WILL propagate to all others. While this is a powerful tool, I find it does not play particularly nicely with isolate scopes. Since isolate scopes read off of the directive's attributes, relying on the attrs object tends to blow that out of the water. As such, this is not a practice I hold in particularly high regard, especially that everything it achieves can be similarly achieved through other means.

#5 Conclusion/Summary
Isolate Scopes are awesome. They are the secret sauce that lets your directives stand alone.  
Transclusion lets you pull in custom widgets and have your DOM be flexible.
Directive Controller's are an excellent way to expose universal APIs.  
Use `$inject` to pull in flexible data sources.
