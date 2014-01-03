---
layout: docs
title: Scopes
prev_section: querying
next_section: eager_loading
permalink: /scopes/
---

## Scope Definitions

NoBrainer allows named scopes to be defined on Models.
Scopes are defined by declaring a class method on a model. Such method must
return a criterion. Example:

{% highlight ruby %}
class Model
  def self.active
    where(:active => true)
  end
end
{% endhighlight %}

You can also use the `scope` method which defines a class method with
whatever you pass as a block. Example:

{% highlight ruby %}
class Model
  scope :active { where(:active => true }
end
{% endhighlight %}

When a class method is found in a chain of criteria, the method is executed and
the returned criterion is merged with the chain. Example with the previous
declared scope:

{% highlight ruby %}
Model.where(:email => /@gmail/).active.count
# Equivalent to
Model.where(:email => /@gmail/).where(:active => true }.count
{% endhighlight %}

## Default Scopes

Default scopes are scopes that are merged in criteria when constructing a query.
Adding `unscoped` in the criteria will disable the use of the default scope.
Adding `scoped` will add it back. Example:

{% highlight ruby %}
class Model
  default_scope { where(:active => true) }
end

Model.count # returns only active models
Model.unscoped.count # returns all models
Model.unscoped.scoped.count # returns active models
{% endhighlight %}

Default scopes are applied at the beginning of the chain when building a query.
The following example illustrates this fact:

{% highlight ruby %}
class Model
  default_scope { order_by(:created_at) }
end

Model.each { } # ordered by created_at
Model.unscoped.order_by(:id).scoped.each { } # ordered by ids, even though scoped is called after
{% endhighlight %}

Unlike the ActiveRecord behavior, `where()` filters in default scopes are not overridable.
For example, consider a Model with a default scope of `where(:active => true)`.
With ActiveRecord, `Model.where(:active => false)` will yield all inactive
models, while NoBrainer will return nothing. This is because NoBrainer will
evaluate `Model.where(:active => true).where(:active => false)`.

## Default Scopes are sometimes ignored

NoBrainer will not use the default scopes in the following cases:

1. When using a `dependent` option on an association, such as  
   `has_many :stuff, :dependent => :destroy`, the Stuff model is used unscoped.
   For example, upon destroy, NoBrainer will perform a `Stuff.unscoped.destroy_all`.
2. When using `validates_uniqueness_of :some_field`, NoBrainer will not use the
   default scope to find any duplicates of `some_field`.
3. When using `find()` or `find!()`.