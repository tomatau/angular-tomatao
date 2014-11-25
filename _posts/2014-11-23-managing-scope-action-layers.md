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

Well - we want to keep the logic of state separate to the logic of fetching, this would allow us to change the persistence method easily. E.g. We might want to use IndexedDb for development or different API end-points for testing and production. Perhaps local storage isn't an option for certain clients so having this logic separate would make this easily configurable without cluttering our state.

Ok - so that's pretty nice but you could say well YAGNI (you ain't gonna need it), and this sort of requirement isn't appropriate for all large projects. Well there's more reasons to split this logic -

Say your API endpoint for getting the profile is actually populated by the same endpoint that populates your `User` model -- endpoints can be very closely coupled but that doesn't mean our Models need the same coupling!! Keeping the fetch logic out of here lets us populate each of these state services independently and not couple them inside each other!

When we start looking at the process of saving and modifying data in our persistence layers, the above reasons are amplified greatly! 

**One fantastic benefit** we get from keeping this fetch separate, is that we can make pipe chains with the fetched data to populate specific subsets in our state layer! E.g. you might fetch the complete user list from your database, you can then store all of this information in the `UserList` model (in the state layer) -- then you can filter a subset and directly populate an `ActiveUserList` state model as well as a `CurrentUserListPage` state model as well! Nice!

#### Why go to all the trouble of making a factory just for that?

Well -- firstly, it's not really *that* much trouble, it's a tiny file and it' quite easy to make -- surely, every developer's heard the phrase "Programs should do one thing and do it well"... why treat our Angular services any differently? :)

Small code blocks are easy to test and maintain, highly reusable and often quite semantic and composable!

What I consider the most important reason of all the above... is that these services now strictly reflect our business logic - they're speaking business use cases way more than a method lost in a 200 line controller.

## So What's an Action Layer again?

An action layer is yet another layer of services that now is responsible for setting and updating our state layer! This can be called a 'Use Case Layer', 'Service Layer', 'Interactors' or 'Controls' so the name isn't important.

In the Flux architecture you would have actions that can modify stores, this is a **very** similar concept. We're just replacing the dispatcher with developer discipline and smaller stores by letting the actions manipulate them directly. 

### One Direction Data Flow

As in Flux, we are making use of a one direction data flow -- Views listen to the State Layer -> controllers, routes and sockets use Actions in -> Actions update the State Layer. It provides a single point of truth for your application state, making it not only easier to manage, but also easier to test!

### Action Services Return Promises

This is a practice I like to belligerently enforce as it allows many great features, such as using actions in resolve blocks and having views respond to successful transactions. Simple but effective! The controllers still remain very small!

## Saving A State Model

So following on from our controller for the user profile, let's have a look at it again.

```js
.controller('UserProfileCtrl', function(
    Profile
){
    this.profile = Profile.get();
})
```

So the corresponding view would populate a form with the profile information and need a submit method.

```html
<form ng-submit="vm.submit()">
    <!-- the ng-model is bound to the Profile state object -->
    <input type="text" name="username" ng-model="vm.profile.name">
    <button type="submit">Submit</button>
</form>
```

Let's add the submit method to the view along with the `Action Layer Service` named `updateUserProfile`.


```js
.controller('UserProfileCtrl', function(
    Profile, updateUserProfile
){
    this.profile = Profile.get();
    this.submit = updateUserProfile;
})
```

Pretty straight forward! We could add things like a model here by wrapping the `updateUserProfile` function call inside a function and calling `then` off the promise that the action returns. A nice small controller and the logic for saving it is completely reusable -- so we can even let our sidebar controller update the profile without ever needing to see the Profile state itself -- and no messy `$scope.$on` listeners!

For good measure, let's have a look at the `updateUserProfile` action, it follows the same routine as the fetchUserProfile action from earlier.

```js
.factory('updateUserProfile', function(
    Profile,
    profileGateway,
    notify,
    analytic
){
    return function updateUserProfile(){        
        return profileGateway.update(Profile.get()) 
            .then(function(updatedProfile){
                // send off some analytics data for monitoring
                analytic("updatedProfile", updatedProfile);
                // submit a notification
                notify("Profile successfully updated!");
                // if they changed their own profile
                // update the current user state object
                // this might change the "Welcome 'username'" in the navbar
                if (Profile.get('user-id') === User.get('id')) {
                    User.update(Profile.getUserData());
                }
            });
    };
})
```

Can you see how this reflects our business use case? We've got updates to analytics and notifications with 8 lines of code! Completely reusable and not even coupled to a persistence abstraction. I think that's pretty awesome for a days work and to top it off we're even updating all of our user interface in one function without a single `$watch` or `$on`!

### Next Post

I will post about further solutions to the problems discussed in [Part 1][part1] and the subject of **.

[part1]: /blog/managing-scope-state-layers/

