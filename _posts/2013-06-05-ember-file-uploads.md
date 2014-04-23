---
layout: post
title:  "From Rails to Ember: File handling"
date:   2013-06-05 17:06:25
categories: ember rails file-handling
---

EDIT: This was something I wrote after a week or two of using Ember. It's pretty low-level, but I'll leave it for the sake of posterity.

***

## Preface

I've recently started testing Ember on an internal web application we're building. Our apps were previously in more vanilla Ruby on Rails, but complicated HTML forms were sometimes slow to render, and as I found myself writing large amounts of jQuery to partially load content through ad hoc AJAX, things started to look ugly and maintainable, and it became increasingly obvious that a more rigorous client-side framework was the way to go. At RailsConf this year I heard Yehuda Katz criticize the view that Javascript is just something you "sprinkle" on your Rails app, without much of a theory of how Javascript should be structured -- I'm trying to take that to heart.

But when you move to something like Ember, a lot of features that used to be easy suddenly become hard. Here I want to document some of the new techniques and workarounds we've used to rebuild some of this lost functionality. To start out, let's talk about file uploads. (Note that all Ember code here is in CoffeeScript, with templates in [Emblem.js](http://emblemjs.com/); the Rails templates are Haml.)

## File uploads in Rails

We use <a href="http://rubygems.org/gems/paperclip">Paperclip</a> for file uploads. It's great. If you have a model called (say) `report` with an associated model called `attachment`, and install Paperclip on the `attachment` model, everything becomes easy.

For upload:

{% highlight haml %}
= file_field_tag "attachment"`
{% endhighlight %}

or for a download link:

{% highlight ruby %}
link_to [@report, attachment] do
  = attachment.electronic_document_file_name
{% endhighlight %}

The attachment controller doesn't have to do much more than this:

{% highlight ruby %}
def show
  attachment = Attachment.find params[:id]
  send_file attachment.electronic_document.path,
            type: attachment.electronic_document_content_type
end
{% endhighlight %}

## File downloads with Ember

The downloads are still very easy in Ember. You can still use the same route on your Rails attachments controller to handle sending the uploaded files. You just have to build an Ember model to represent your attachment, something like this:

{% highlight coffeescript %}
App.Attachment = DS.Model.extend
  report: DS.belongsTo 'App.Report'
  notes: DS.attr 'string'
  createdAt: DS.attr 'date'
  updatedAt: DS.attr 'date'
  electronicDocumentFileName: DS.attr 'string'
  electronicDocumentFileSize: DS.attr 'number'
{% endhighlight %}

And then you may want to build an Ember controller (remembering that Ember controllers are really just like presenter objects) that adds some computed property to generate a download path for each attachment:

{% highlight coffeescript %}
App.AttachmentController = Ember.ObjectController.extend
  downloadPath: (->
    fileName = @get 'electronicDocumentFileName'
    if fileName and fileName.length
      id = @get 'id'
      App.RailsRootPath + "/download/#{id}"
    else
      null
  ).property()
{% endhighlight %}

(In our case, we deploy our apps at different paths depending on the environment, so I define App.RailsRootPath in an Ember initializer.)

Once you've set this up, it's easy to create file download links anywhere in the app. Something like:

{% highlight haml %}
a href=attachment.downloadPath
  = attachment.electronicDocumentFileName
{% endhighlight %}

If you set up a download route in your Rails routes that points to `attachments#show`, things will work perfectly.

Unfortunately, uploads are slightly less straightforward. [EDIT: The approach to uploading [described two years ago by Hedtek](http://devblog.hedtek.com/2012/04/brief-foray-into-html5-file-apis.html) is still basically correct, and is recommended.]