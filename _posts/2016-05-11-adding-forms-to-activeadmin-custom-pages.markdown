---
layout: post
title:  "Adding forms to ActiveAdmin custom pages"
date:   2016-05-11 15:52:18 +0300
tags: [rails]
---

[ActiveAdmin][activeadmin] is an administration framework for Ruby on Rails applications. It allows to easily create admin pages to manage rails entities, offering out-of-the-box CRUD functionality.
Sometimes, however, you need some bit of funcionality that is out of scope of a single resource.
<!-- more -->

The solution is to use [custom pages][activeadmin-custompages], which are not tied to
a single entity. The documentation explains how to add content and actions to custom page, but not forms. In my case, I had to create a custom page containing a very simple form,
with a single text-field input and a submit button. Once the button is clicked, some action that depends on the value in the form had to take place.


While ActiveAdmin uses [Formtastic] for forms generation in the create/edit pages, it is not available in custom pages. Instead, you have to use [Arbre] to generate the form HTML. This also means
that you may not use Formastics's constructs such as `f.inputs` or `f.actions` in your form.

See below for a sample source code:

{% highlight ruby %}
ActiveAdmin.register_page "my_custom_page" do

  page_action :foo, method: :post do
    # do some logic using params['my_field']
    redirect_to "/"
  end

  content do
    form action: "my_custom_page/foo", method: :post do |f|
      f.input :my_field, type: :text, name: 'my_field'
      f.input :submit, type: :submit
    end
  end

end
{% endhighlight %}

[activeadmin]: http://activeadmin.info
[activeadmin-custompages]: http://activeadmin.info/docs/10-custom-pages.html
[Formtastic]: https://github.com/justinfrench/formtastic
[Arbre]: https://github.com/activeadmin/arbre
