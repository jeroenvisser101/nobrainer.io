---
layout: docs
title: Querying
prev_section: associations
next_section: scopes
permalink: /querying/
---

NoBrainer supports a rich query language that supports a wide range of features.
This section is organized in two parts. The first one describes the methods
can be chained to build a query, the second one describes the methods that
finishes and terminate a query.

## Chainable Criteria

### all

`Model.all` yields a criterion that can be chained with other criteria.
This is not so useful as most of the chainable criteria and terminators can be
directly called on the Model class. For example `Model.each { }` will enumerate
all models.

### where()

The `where()` method selects documents for which its given predicates are true.
The rules are the following:

* `where(p1,...,pN)` returns the documents that matches all the predicates `p1,...,pN`.

* `where(p1).where(p2)` is equivalent to `where(p1, p2)`

The predicates are described below:

* `[p1,...,pN]` evaluates to `:and => [p1,...,pN]`.
* `:and => [p1,...,pN]`: evaluates to true when all the predicates are true.
* `:or => [p1,...,pN]`: evaluates to true when at least one of the predicates is true.
* `:not => p`: evaluates to true when `p` is false.
* `:attr => value` evaluates to `:attr.eq => value`
* `:attr.eq => /regexp/` evaluates to true when `attr` matches the regular expression.
* `:attr.eq => (min..max)` evaluates to true when `attr` is between the range.
* `:attr.eq => value` evaluates to true when `attr` is equal to `value`.
* `:attr.not => value` evaluates to `:attr.ne => value`.
* `:attr.ne => value` evaluates to `:not => {:attr.eq => value}`.
* `:attr.gt => value` evaluates to true when `attr` is greater than `value`.
* `:attr.ge => value` evaluates to true when `attr` is greater than or equal to `value`.
* `:attr.gte => value` evaluates to `:attr.ge => value`
* `:attr.lt => value` evaluates to true when `attr` is less than `value`.
* `:attr.le => value` evaluates to true when `attr` is less than or equal to `value`.
* `:attr.lte => value` evaluates to `:attr.le => value`.
* `:attr.in => [value1,...,valueN]` evaluates to true when `attr` is in the specified array.
* `lambda { |doc| rql_expression(doc) }` evaluates the RQL expression.

A couple of notes:

* `:attr.keyword => value` can also be written as `:attr.keyword value`.

* When using the `in` keyword, NoBrainer will use the [`get_all()`](http://www.rethinkdb.com/api/ruby/#get_all)
command if there is an index declared on `attr`. Otherwise, NoBrainer will
construct a query equivalent to `Model.where(:or => [:attr => value1,...,:attr => valueN])`.

* The comparison operations could leverage an index and use the RQL
[`between()`](http://www.rethinkdb.com/api/ruby/#between) command, but this feature is not implemented yet.

* `where()` will try to use one of your declared indexes for performance.
Learn more about indexes in the [indexes section](/docs/indexes).

* `Model.where(:attr => value1).where(:attr => value2)` will match no
documents if `value1 != value2`, even when using a `default_scope`.

* `where()` can also take a block to specify an additional RQL filter.

* Nested hash queries are not yet supported. Use a RQL filter in this case.

As an example, one can construct such query:

{% highlight ruby %}
Model.where(:or => [->(doc) { doc[:field1] < doc[:field2] },
                    :field3.in ['hello', 'world'])
     .where(:field4 => /ohai/, :field5.gt 4)

{% endhighlight %}

### order_by()/reverse_order

`order_by()` allows to specify the ordering in which the documents are returned.
Below a couple of examples to show the usage of `order_by()`:

* `order_by(:field1 => :asc, :field2 => :desc)` orders by field1 ascending
  first, and then field2 descending.
  This syntax works because since Ruby 1.9, hashes are ordered.
* `order_by(:field1, :field2)` is equivalent to `order_by(:field1 => :asc, :field2 => :asc)`
* `order_by { |doc| doc[:field1] + doc[:field2] }` sorts by the sum of field1
  and field2 ascending.
* `order_by(->(doc) { doc[:field1] + doc[:field2] } => :desc)` sorts by the sum
  of field1 and field2 descending.
* criteria.reverse_order yields criteria with the opposite ordering.
* The latest specified `order_by()` wins. For example,
  `order_by(:field1).order_by(:field2)` is equivalent to `order_by(:field2)`.

Note: NoBrainer always `order_by(:id => :asc)` by default.

### skip()/offset()/limit()

* `criteria.skip(n)` will skip `n` documents.
* `criteria.offset(n)` is an alias for `criteria.skip(n)`.
* `criteria.limit(n)` limits the number of returned documents to `n`.

### raw

* `criteria.raw` will no longer output model instances, but attribute hashes as
received from the database.

### scoped/unscoped

* `criteria.scoped` will enable the use of the default scope on the model if
defined. This behavior is the default.
* `criteria.unscoped` will disable the default scope.

### with_index/without_index/used_index/indexed?

* `criteria.with_index(index_name)` forces the use of index_name during the where() RQL
generation. If the index cannot be used, an exception is raised.
* `criteria.without_index` disables the use of indexes.
* `criteria.used_index` shows the index name used if any.
* `criteria.indexed?` returns `true` when an index is in use.

### with_cache/without_cache

* `criteria.with_cache` enables the use of the cache.
* `criteria.without_cache` disable the use of the cache.
* `criteria.reload` kills the cache.

Read more about caches in the [caching section](/docs/caching).

### includes()

* `criteria.includes(:some_association)` eager loads the association. Read more
about eager loading in the [eager loading section](/docs/eager_loading).

## Terminating a Criteria

### count

* `criteria.count` returns the number of documents that matches the criteria.
* `criteria.empty?` is an alias for `count == 0`
* `criteria.any?` is an alias for `count != 0` if no block is given. When a block
is given, the method is sent to `to_a`.

### update_all/replace_all/delete_all/destroy_all

* `criteria.update_all` update all documents matching the criteria. Returns the
number of replaced documents (which can be different from the number of matched
documents).
* `criteria.replace_all` replaces all documents matching the criteria. Returns the
number of replaced documents.
* `criteria.delete_all` deletes all documents matching the criteria. Returns the
number of deleted documents.
* `criteria.destroy_all` instantiate the models, run the destroy callbacks and
deletes the documents. Returns the array of destroyed instances.

### each/to_a

* `criteria.each` enumerates over the documents.
* `criteria.to_a` returns an array of documents matching the criteria.
* `criteria.some_method_of_array` will proxy `some_method_of_array` to `to_a`

### first, first!, last, last!

* `criteria.first` returns the first matched document.
* `criteria.last` returns the last matched document.
* `criteria.first!` returns the first matched document, raises if not found.
* `criteria.last!` returns the last matched document, raises if not found.

The bang versions raise a `NoBrainer::Error::DocumentNotFound` exception if not found
instead of returning `nil`.

### inc_all/dec_all

* `criteria.inc_all(field, value)` increments all matched document field by value.
* `criteria.inc_all(field)` is an alias for `criteria.inc_all(field, 1)`.
* `criteria.dec_all` is an alias for `criteria.inc_all`.

Are these increment methods useful? Perhaps not. Post an issue on Github to
express yourself.

### Manipulating Criteria

You can merge two criteria with `merge()` or `merge!()`:

{% highlight ruby %}
criteria3 = criteria1.merge(criteria2)
criteria1.merge!(criteria2)
{% endhighlight %}

To retrieve the criteria of a model instance, you may use `instance.selector`.
This is a wrapper for `Model.unscoped.where(:id => instance.id)`.