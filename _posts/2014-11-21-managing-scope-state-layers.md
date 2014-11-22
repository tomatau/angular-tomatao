---
layout: post
title:  "Managing Scope: #1 State Layers"
date:   2014-11-21 12:00:00
categories: angularjs scope state models
---

## Scope gets Complicated

One of the most common problems I hear people running into is that scope gets out of control. When this happens, files can become very large and complicated to deal with the edge cases, such as: page refreshes, multiple views needing to respond to events, route organisation and persisting state! There are of course, many other potential problems that can pop up.

I will first talk about some existing solutions and their problems. Finally, I will talk about separating state from behaviour using a state layer that reduces controller boilerplate, increases code-reuse and provides the facility to manage persistence and page loads effectively.

I will no doubt make a few more blog posts about the subject of `state` management in AngularJS.

## Existing Solutions

There are a number of existing solutions to manage this complexity - I'll cover the one's that often pop up!

#### 1. Pass by Reference on Scope

This is a very basic fix, when working with `$scope`, if you add value properties directly such as *Strings* or *Numbers* -- updates may not behave as you expect. JavaScript is "call by sharing" language -- so use objects to maintain references and keep those bindings in sync!

```javascript
.controller('MyCtrl', function($scope){
    $scope.value = "Any old value"; // Bad! Potentially problematic!
    $scope.reference = {}; // Good! Pass by reference giving you more control.
})
```

**Problems**

- This is merely a best practice and doesn't address managing state.

#### 2. Controller Alias Syntax

You can avoid the use of `$scope` greatly by aliasing your controllers inside each view which is a fantastic way to reduce complexity. As a result, nested controllers become far more manageable as their $scope's do not collide.

Originally, you would use the `$scope` to add data to the view, like so:

```js
.controller('MyCtrl', function($scope){
    $scope.data = {};
})
```

With the view like so:

```html
<div ng-controller="MyCtrl">
    {% raw %}{{ data }}{% endraw %}
</div>
```

Using an alias for the controller you can remove the need for scope and just use the `this` of your controller. 

```js
.controller('MyCtrl', function(){ // no scope
    this.data = {};
})
```

This can be enabled on a simple view by adding the `as [controller's alias]` in the controller attribute and referencing the alias which will store the state.

```html
<div ng-controller="MyCtrl as my">
    {% raw %}{{ my.data }}{% endraw %}
</div>
```

Alternatively you can add it to you `$routeProvide.when` configuration objects or directive definition objects by using the *controllerAs* property:

```js
controllerAs: 'my', // set to whatever alias you would like
controller: 'MyCtrl'
```

**Problems**

- This again is simply a best practice for nesting controllers.
- We are still storing data on $scope but on a property inside the scope object, it adds "the dot" to our expressions but doesn't address the problems we'v mentioned such as "initial state", "events", "route hierarchies" or "persisting data".
- We have more boilerplate in views, routes and directives instead of in the controllers themselves.
- Often you still need to inject `$scope` anyway for watches, or events.
- Often need to save assign `this` to another name to avoid context bugs e.g. `var vm = this`, to make `$scope.$watch` functions work, etc...

#### 3. Persistence Abstractions

Taking the state outside of the controller is a great practice, it can reduce the size of controllers greatly -- making them both easier to manage and easier to test! One very common solution here, is to make factory or service methods which make specific $http calls to fetch the state. You can then expose the methods to controllers.

These can be called "models", "entities", "services", "resources", "repositories" and many other names... I will just call them Persistence Abstractions as the other names have loaded meanings.

Here's an example of a very simple one for managing a user:

```js
.factory('User', function($http, $q, userEndpointConfig){
    var userData = {};
    return {
        syncUser: function(){
            return $http(userEndpointConfig)
                .success(function(data){
                    userData = data;
                });
        },
        getUser: function(){
            var def = $q.defer();
            if (userData.length) { def.resolve(userData); }
            else {
                this.syncUser().then(function(){
                    def.resolve(userData);
                });
            }
            return def.promise;
        }
    };
})
```

Now, multiple controllers can use the `User.getUser()` method to populate their scope and all refer to the same object. This is very helpful for code reuse. You can even make basic implementations that other Persistence Abstractions can extend such as *ngResouce* and *RestAngular*.

**Problems**

- This alone can introduce many promises into controllers which are hard to test.
- Any old controller may be updating this Persistence Abstraction, introducing problems of responsibility.
- Which controller should be setting it's initial state on a page refresh? What about for each route? Having an encompassing controller for each route feels messy and can be very difficult to test.
- The "models" become very large -- managing both behaviours, state and persistence.
- Swapping out one backend for say indexedDB, localStorage, Web Workers, sockets or Firebase, etc... could be very time consuming and problematic -- having to change each method throughout each Persistence Abstraction.

#### 4. Resolve Blocks

When you're configuring route's through the `$routeProvider` or [Ui Router][ui.router]'s `$stateProvider` -- each `when` or `state` has a `resolve` property that can be used to populate the state on a page load. This is great in conjunction with Persistence Abstractions as you can remove much of the promise logic from your controllers. Thus, much easier to test!

Using [Ui Router][ui.router] makes this even better, as we can use nested states to manage these resolves for multiple controllers per route! More to come on this soon!

```js
$routeProvider
    .when('/user-profile', {
        resolve: {
            user: function(User){
                return User.getUser();
            }
        }
    });
```

We can also use resolve blocks to help with our initial page load. We're also moving the calls to our persistence outside of the controller, making it much smaller.

**Problems**

- When navigating between routes, it isn't explicit about how many times these methods will be called, e.g. if you go back to the previous page -- will it be called again or will it use a cache of the previous result? It isn't obvious from looking at it.
- Still end up with big Persistence Abstractions.
- Views still can't easily respond to each other's events.

#### 5. Event Based Behaviours

To tackle the problem of responding to events, people often subscribe callback functions to specific events in their controllers. This is fairly straight forward and quite effective for loose coupling your components such as analytical bookkeeping, notifications any whatever else! Events are incredibly powerful for creating resilient and scalable systems.

```js
.controller('MyCtrl', function($scope){
    $scope.$on('namespaced:custom-event', function(){
        // respond to event
    });
})
```

**Problems**

- Event's in angular can be a little bit confusing as they inherit from parent scopes which you cannot see by looking at a single controller.
- $rootScope's injected into controllers aren't the parent scope of controller's scopes and can be confusing.
- When the number of events grow in number, the boilerplate starts becoming tedious and messy, controllers become littered with `$scope.$on`.
- Doesn't provide any hierarchy for choosing which place to place initialisation logic.
- As controllers require compilation for testing, they aren't the ideal place to unit test event based behaviours.

#### 6. State Machines

State machines are an incredibly useful pattern for organising and having some pattern to manage different application states. This can be conjoined effectively with routing to manage page refreshes effectively. One great tool for this is [Ui Router][ui.router] -- this not only allows you to manage URL based routes, but also lets you manage *states* and their dependencies.

[Ui Router][ui.router] has additional features on top of the normal ngRouter module such as nested routes, named states and the potential to build layouts for specific states that can have optional resolve properties.

This is incredibly powerful when mixed with resolve blocks and Persistence Abstractions for controlling your initial page load. Giving hierarchies to our states and allow

**Problems**

- Still no way to manage events that trigger behaviour across multiple views.
- Can end up with very large configurations of routing and states that become difficult to manage.

# Using a State Layer

So how can we manage these two remaining problems? The overall solution involves a number of techniques:

- A State Layer
- An Action Layer
- One direction data flow
- Resolve hierarchies and pipes
- Initial state control
- Appropriate module splits and independent configurations
- Boundary control
- Inversion of Control dependency injection

In this post I'll only chat about a **state layer** - the basic idea is to have a layer of "Models" in the more traditional sense of a "model". These can be services or factories that simply hold some state - very similar to the Persistence Abstractions we had before but reducing their complexity by keeping all persistence calls separate! 

When you separate out the calls to `$http`, `localStorage` or any other  persistence layer from the models which hold their item state -- you are focusing on single responsibilities, reducing boilerplate, increasing reuse and making your overall model much more simple.

## Objects Keeping References

When you use a state layer, it can be helpful to be strict about keeping an object or array that maintains it's initial instance. This will ensure that any views using this Model are always pointing at the same object!

```js
.factory('Model', function(){
    function Model(){
        this.data = {};
    }

    Model.prototype.set = function(modelData, value){
        if (typeof modelData == "string")
            this.data[modelData] = value;
        else
            angular.extend(this.data, modelData);
    };

    Model.prototype.get = function(prop) {
        if (prop == null) return this.data;
        return this.data[prop];
    };

    Model.prototype.reset = function() {
        // keep reference but clear all properties
        for (var member in this.data) delete this.data[member];
    };
})
```

Here we have a basic factory that can be responsible for a certain state. Note the reset method that will remove all of the data object's properties without changing the objects reference.

Now our controller can simply pass the object to it's view and be sure that it will reflect the current state.

```js
.controller('MyCtrl', function(Model){
    this.data = Model.get();
})
```

Testing of our entities also now is not tied to any specific endpoint, easy tests! Also, if we wish to change our persistence URL or even change our persistence method -- we're free to do so without needing to change our models. These Models only have **one concern**, and that is **state**!

```js
describe('Model', function () {
    var modelData = { id: 123, name: 'TEST001' };

    beforeEach( module("models") );

    it('should start empty', inject(function(Model) {
        expect(Model.get()).toEqual({});
    }));

    it('should get the updated object', inject(function(Model) {
        var m = Model.get();
        m.id = Model.id;
        expect(Model.get()).toBe(m);
    }));

    it('should provide a set method', inject(function (Model) {
        Model.set(modelData);
        expect(Model.get()).toEqual(modelData);
    }));
});

```

## Extending

Backbone JS has a very similar concept in Collections and Models but the backbone style also involves the RESTful API calls. When you remove the persistence layer integration, having base entities to extend becomes far more practical. You can even use the Backbone's `extend` method to provide a nice baseModel and baseEntity.

```js
.factory('extend', function(){
    return function(protoProps, staticProps) {
        var parent = this,
            child;
        if (protoProps && _.has(protoProps, 'constructor')) {
            child = protoProps.constructor;
        } else {
            child = function(){ return parent.apply(this, arguments); };
        }
        _.extend(child, parent, staticProps);
        var Surrogate = function(){ this.constructor = child; };
        Surrogate.prototype = parent.prototype;
        child.prototype = new Surrogate();
        if (protoProps) _.extend(child.prototype, protoProps);
        child.__super__ = parent.prototype;
        return child;
    };
});
```

Add the method to your BaseCollection and BaseModel like so:

```js
.factory('BaseCollection', function(extend){
    function BaseCollection(){
        this.list = [];
    }
    BaseCollection.prototype.reset = function() {
        this.list.length = 0;
    };
    // ...
    BaseCollection.extend = extend;
    return BaseCollection;
})
```

And now all of your collections and models can easily be extended and reused!

```js
.factory('UserList', function(BaseCollection){
    var UserList = BaseCollection.extend({
        findActive: function(){
            return _.where(this.get(), {
                status: "active"
            });
        }
    });
    return new UserList();
});
```

This is a very flexible approach as we can make new models and collections very easily to manage independent lists or entities without affecting the state of original collections and models. Also with small models, the number of tests can be small too - simply test the implementation of our well tested base object.

- Now our resolves can populate this state layer and we have taken control of state without tying it to the complexities of a persistence layer that will often involve transactions.
- We have a reusable and easily testable units that many views can listen to through simple object references.
- We're free to change persistence and extend specific behaviours very easily.
- Scope is no longer responsible for state - we have a special layer of units that all other controllers and views can refer to.

**Problems**

- This still doesn't solve the problem of setting an initial state after reloading a page for example or route changes... but at least we have a place to put it!
- These units are now potentially depended on by **all** controllers in our application so can be quite fragile.
- These Models are also very similar to the idea of React stores... the one key difference is that our models have public set methods. This can be a problem and will require discipline to keep manageable --
- To solve these problems, we should talk about an Action layer, one direction data flow.

#### Next Post

I will next post about solutions to the above problems using Action layers and one direct data flows.

[ui.router]: https://github.com/angular-ui/ui-router


