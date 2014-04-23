---
layout: post
title:  "Modal presenter objects in Ember"
date:   2014-04-23 08:32:31
categories: ember modals
---

OK, a year has gone by, Ember has matured a lot, released version 1.0 (though Ember Data is still in beta), the first EmberConf happened and it was pretty interesting, and the community is growing, in spite of the ongoing [flamewars about the very idea of Javascript frameworks](https://news.ycombinator.com/item?id=7633175). Some parts of Ember are very well developed. Others still don't have such clear standards.

I've been thinking about modal dialogs a bunch lately. Our most recent new application has about nine of them, which is more than enough to want a clean way of doing things. Ember does have a pretty straightforward paradigm for [rendering modals into your application](http://emberjs.com/guides/cookbook/user_interface_and_interaction/using_modal_dialogs/), which I've found useful. There's a global modal outlet, managed by the `ApplicationRoute`, and you can call actions on the `ApplicationRoute` to show or hide your modals. The docs say to use a component; when I first started writing these, we used views, so I've stuck with those for our code.

But one thing struck me as still not that clear. How do you bind data to your modals? The guides suggest that, if you want to let someone edit, say, a `name` field on a `record`, you'd just bind a text field in your modal to something like `record.name`, and, presto, all the binding is set up.

But what if you want to have a cancel button on your modal that aborts the local changes to the `record`? The simple version of data binding doesn't cover that. You need to have a way of *provisionally* changing a field and then *confirming* your changes (say, with an OK button).

There are various ways you can do this. You can use the [rollback](http://emberjs.com/api/data/classes/DS.Model.html#method_rollback) method from Ember Data to handle a cancel button, but it's not really the right tool for the job, because it rolls back *all* changes to your `record`, not just the ones made most recently in a modal dialog. You can make a duplicate of `record` and bind the modal to that, but then you would need to figure out how to merge your changes back to `record` when the user clicks OK. You could wish for more intricate versioning or undo features... but those aren't really built into Ember Data at this point.

Instead of any of those options, I ended up introducing a new intermediate class to bind to in my modals. Call it `ModalPresenter`. It understands that it has a backing object, which I'll call `parent`; it has a list of fields that it should copy from the parent object and make available to the modal dialog; and it has a `saveToParent` method that we can invoke when a user clicks OK.

Here's the code:

{% highlight coffeescript %}
# Generic modal presenter
# It assumes that parent and allowedFields will be set on creation.

App.ModalPresenter = Ember.Object.extend
  init: ->
    @allowedFields.forEach (field) =>
      @set field, @parent.get(field)

  saveToParent: ->
    @allowedFields.forEach (field) =>
      @parent.set field, @get(field)

{% endhighlight %}

You use it like this from your controller. (I'll leave out the actual modal views themselves, since that part is pretty much stock Ember. Note that my modals have a `modalContent` object that they use to display data.)

{% highlight coffeescript %}
App.TaskIndexController = Ember.ArrayController.extend
  launchTaskDialog: (task) ->
    modalContent = App.ModalPresenter.create
      parent: task
      allowedFields: ['assignee']  # or whatever fields you need
    @send 'openModal', 'assignTask', modalContent

  updateTaskDialog: (modalView) ->
    modalView.get('modalContent').saveToParent()
    modalView.send 'close'
{% endhighlight %}
