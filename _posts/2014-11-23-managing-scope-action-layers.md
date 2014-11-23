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

These are great projects, but trying to understand all of the ins and outs of these controllers would take me hours -- I'd also feel very nervous making any changes!

## TLDR; Part 1

You can take a look at [part 1 here][part1] -- I spoke about a few techniques that people already use for reducing `$scope` complexity.

1. Avoiding values on `$scope` with objects
2. Controller As syntax
3. Persistence abstractions
4. Resolve blocks
5. Scope Event
6. State Machines such as `ui.router`

At each of these solutions I also listed a number of problems with each -- and how none of these really address the problems of complex State!

Then I said that the overall solution for these problems requires a combination of several techniques. The first technique was to use a State Layer.

### State Layer

These are basically a group of services whose only concern is to store the current application state (that means no `$http` calls)! These should be read only from the perspective of controllers to avoid sync problems and that will require discipline, so still not ideal!

Read only? Yes that's right -- so who sets their state?


[part1]: /blog/managing-scope-state-layers/