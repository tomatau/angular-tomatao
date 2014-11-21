---
layout: post
title:  "Managing Scope - Using a State Layer"
date:   2014-11-21 12:00:00
categories: angularjs scope state models
---

### Scope gets Complicated

One of the most common problems I head people running into is that scope gets out of control. When this happens, files can become very large and complicated to deal with the edge cases such as: page refreshes, multiple views needing to respond to events, route organisation and persisting state! There are of course, many more potential problems too.

I will first talk about some existing solutions and their problems. Finally, I will talk about separating state from behaviour using a state layer that reduces controller boilerplate, increases code-reuse and provides the facility to manage persistence and page loads effectively.

I will no doubt make a few more blog posts about the subject of `state` management in AngularJS.

## Existing Solutions

There are a number of existing solutions to manage this complexity; here are a few:

### Pass by Reference on Scope

This is a very basic fix, when working with `$scope`, if you add value properties directly such as *Strings* or *Numbers* -- updates may not behave as you expect. JavaScript is a pass by value language -- so use objects to maintain references and keep those bindings in sync.

```javascript
.controller('MyCtrl', function($scope){
    $scope.value = "Any old value"; // Bad! Potentially problematic!
    $scope.reference = {}; // Good! Pass by reference giving you more control.
})
```

**Problems**

- This is merely a best practice and doesn't address managing state.

### Controller Alias Syntax

You can avoid the use of `$scope` greatly by aliasing your controllers inside each view which is a fantastic way to reduce complexity. As a result, nested controllers become far more manageable as their $scope's do not collide.

Originally, you would use the `$scope` to add data to the view, like so:

```js
.controller('MyCtrl', function($scope){
    $scope.data = {};
})
```

With the view like so:

```html
<div controller="MyCtrl">
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
<div controller="MyCtrl as my">
    {% raw %}{{ my.data }}{% endraw %}
</div>
```

Alternatively you can add it to you `$routeProvide.when` configuration objects or directive definition objects by using the *controllerAs* property:

```javascript
controllerAs: 'my', // set to whatever alias you would like
controller: 'MyCtrl'
```

**Problems**

- This again is simply a best practice for nesting controllers.
- We have more boilerplate outside of controllers.
- Often need to save assign `this` to another name to avoid context bugs e.g. `var vm = this`, to make `$scope.$watch` functions work, etc...

### Angular Resources and $http Wrappers

Taking the state outside of the controller is a great practice, it can reduce the size of controllers greatly -- making them both easier to manage and easier to test! One very common solution here, is to make factory or service methods which make specific $http calls to fetch the state. You can then expose the methods to controllers.

These can be called "models", "entities", "services", "resources", "repositories" and many other names... I will just call them Persistence wrappers as the other names have loaded meanings.

Here's an example of a very simple one for managing a user:

```javascript
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

Now, multiple controllers can use the `User.getUser()` method to populate their scope and all refer to the same object. This is very helpful for code reuse. You can even make basic implementations that other Persistence Wrappers can extend such as *ngResouce* and *RestAngular*.

**Problems**

- This alone can introduce many promises into controllers which are hard to test.
- Any old controller may be updating this Persistence Wrapper, introducing problems of responsibility.
- Which controller should be setting it's initial state on a page refresh? What about for each route? Having an encompassing controller for each route feels messy and can be very difficult to test.
- The "models" become very large -- managing both behaviours, state and persistence.
- Swapping out one backend for say indexedDB, localStorage, Web Workers, sockets or Firebase, etc... could be very time consuming and problematic -- having to change each method throughout each Persistence Wrapper.

### Resolve Blocks

When you're configuring route's through the `$routeProvider` or ui.router's `$stateProvider` -- each `when` or `state` has a `resolve` property that can be used to populate the state on a page load. This is great in conjunction with Persistence Wrappers as you can remove much of the promise logic from your controllers. Thus, much easier to test!

Using UI.Router makes this even better, as we can use nested states to manage these resolves for multiple controllers per route! More to come on this soon!

```javascript
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
- Still end up with big Persistence wrappers.
- Controllers still can't so easily respond to each other's events.

### `$scope` and `$rootScope` Events

### State Machines

## Using a State Layer





