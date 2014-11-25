---
layout: post
title:  "Managing Scope: #2 Action Layers"
date:   2014-11-23 21:00:00
categories: angularjs scope state actions models
---

This is part 2 of a discussion about complex `$scope` objects and confusing state. Just to give you a sense of how scopes can quite complicated and difficult to manage -- here are some controllers from large(ish) open source projects after a quick google search

- https://github.com/mjibson/goread/blob/master/static/js/site.js
- https://github.com/coinkite/coinkite-bitcoin-atm/blob/master/project.js
- https://github.com/pikock/bootstrap-magic/blob/master/app/js/controllers.js
- https://github.com/shidhincr/LookAround/blob/master/app/js/controllers.js
- https://github.com/ivan-dyachenko/bamboo-status/blob/master/js/bamboo-status.js

These are great projects, but trying to understand all of the ins and outs of these controllers would take me hours -- I'd also feel very nervous making any changes here!

## TLDR; From Part 1

You can take a look at [part 1 here][part1] -- I spoke about a few techniques that people already use for reducing `$scope` complexity.

1. Avoiding value properties on `$scope` with objects.
2. Controller As syntax for nested controllers.
3. Persistence abstractions.
4. Resolve blocks for routes.
5. Scope Events for triggering behaviours.
6. State Machines such as `ui.router`.

Each of these solutions also had a number of problems with each -- and how none of these really address the problems of complex State fully -- even when using all of them together!

Then I said that the overall solution for these problems requires a combination of several techniques. The first technique was to use a State Layer.

### A State Layer?

These are basically a group of services (factories) whose only concern is to store the current application state (that means no `$http` calls)! These should be read only from the perspective of controllers to avoid sync problems and that will require discipline, so still not ideal!

Read only? Yes that's right -- so who sets them?

## The Action Layer

So you've got some factories that store the application's current state. Let's follow an example of a **user's profile page**. Firstly, we could have a factory that stores the user's profile, called Profile.

```js
.controller('UserProfileCtrl', function(Profile){
    this.profile = Profile.get();
})
```

Looks pretty lightweight so far, cool - so it would make sense to get and set the profile data in a resolve block for this route! (I'm using `ui.router` here.)

```js
.config(function($stateProvider){
    $stateProvider.state('root.user-profile', {
        url: 'user/profile/:id',
        controller: 'UserProfileCtrl',
        resolve: {
            ready: function($routeParams, fetchUserProfile){
                return fetchUserProfile($routeParams.id);
            }
        }
    })
})
```

You see the `fetchUserProfile` service - the idea is to intentionally separate the logic that communicates with persistence (the Back-End, indexedDb, an API, etc...) from the service that stores the application state! This service is going to be very small and concise, returning a promise, being re-usable and potentially serving more than just the fetch -- this could be a whole specific use case!

```js
.factory('fetchUserProfile', function(
    Profile,
    profileGateway
){
    return function fetchUserProfile(profileId){        
        return profileGateway.fetch(profileId)
            .then(function(fetchedProfile){
                Profile.set(fetchedProfile);
            });
    };
})
```

The code above is what will live in our **Actions Layer** - pretty simple and you may be asking 'why go to all the trouble of making a factory just for that?'. You could also ask 'why not just put the fetch on the profile itself?' and even 'why not just do all this in the controller?'!

So let me answer each of these questions - in reverse!

#### Why not just do all this in the controller?

This is a big **NO**!!

Firstly, controllers in Angular are super closely tied to views, putting a fetch on a controller is like making HTTP calls in your view! What if you want to have 5 controllers all access the profile data... now you'll need to wrap all those views in one parent view who makes the fetch and share it's scope with all the children controllers! Can you sense the tight coupling?

Secondly, to test a controller, you need to compile it -- do you really want to be compiling HTML in order to test a $http fetch? Never mind testing promises through that HTML -- it's going to get messy!

Thirdly, what happens when you want to play with your user experience, move some logic and HTML around? It's going to be way more convoluted than it needs to be. Not to mention all of the potential duplicate code!

#### Why not just put the fetch on the profile itself?

We want to keep the logic of state separate to the logic of fetching, 

#### Why go to all the trouble of making a factory just for that?


[part1]: /blog/managing-scope-state-layers/







