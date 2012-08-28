# Power Enum

https://github.com/albertosaurus/power\_enum

Enumerations for Rails 3.X Done Right.

## What is this?:

Power Enum allows you to treat instances of your ActiveRecord models as though they were an enumeration of values.
It allows you to cleanly solve many of the problems that the traditional Rails alternatives handle poorly if at all.
It is particularly suitable for scenarios where your Rails application is not the only user of the database, such as
when it's used for analytics or reporting.

Power Enum is a fork of the Rails 3 modernization made by the fine folks at Protocool
https://github.com/protocool/enumerations\_mixin to the original plugin by Trevor Squires.  While many of the core ideas
remain, it has been reworked and a full test suite written to facilitate further development.

At it's most basic level, it allows you to say things along the lines of:

    booking = Booking.new(:status => BookingStatus[:provisional])
    booking.status = :confirmed
    booking = Booking.create( :status => :rejected )

    Booking.where(:status_id => BookingStatus[:provisional])

    BookingStatus.all.collect {|status|, [status.name, status.id]}

    Booking.with_status :provisional, :confirmed

See "How to use it" below for more information.

## Installation

Add the gem to your Gemfile

    gem 'power_enum'

then run

    bundle install

## Gem Contents

This package adds:
- Two mixins and a helper to ActiveRecord
- Methods to migrations to simplify the creation of backing tables
- A generator to make generating enums easy
- Custom RSpec matchers to streamline the testing of enums and enumerated attributes (Since version 0.6.0)

`acts_as_enumerated` provides capabilities to treat your model and its records as an enumeration.
At a minimum, the database table for an acts\_as\_enumerated must contain an 'id' column and a column
to hold the value of the enum ('name' by default).  It is strongly recommended that there be
a NOT NULL constraint on the 'name' column.  All instances for the `acts_as_enumerated` model
are cached in memory.  If the table has an 'active' column, the value of that attribute
will be used to determine which enum instances are active.
Otherwise, all values are considered active.

`has_enumerated` adds methods to your ActiveRecord model for setting and retrieving enumerated values using an
associated acts\_as\_enumerated model.

There is also an `ActiveRecord::VirtualEnumerations` helper module to create 'virtual' acts\_as\_enumerated models
which helps to avoid cluttering up your models directory with acts\_as\_enumerated classes.

## How to use it

In the following example, we'll look at a Booking that can have several types of statuses, encapsulated by BookingStatus enums.

### generator

Invoke the generator to create a basic enum:

`rails generate enum booking_status`

You should see output similar to this:

    create  app/models/booking_status.rb
    create  db/migrate/20110926012928_create_enum_booking_status.rb
    invoke  test_unit
    create    test/unit/booking_status_test.rb

That's all you need to get started.  In many cases, no further work on the enum is necessary.  You can run `rails generate enum --help`
to see a description of the generator options.  Notice, that while a unit tests is generated by default, a fixture isn't.  That is because
fixtures are not an ideal way to test acts\_as\_enumerated models.  I generally prefer having a hook to seed the database from seeds.rb
from a pre-test Rake task.

### migration

If you're using Rails 3.0, your migration file will look something like this:

    class CreateEnumBookingStatus < ActiveRecord::Migration
    
      def self.up
        create_enum :booking_status
      end
      
      def self.down
        remove_enum :booking_status
      end
    
    end
    
If you're using Rails 3.1 or later, it will look something like this:

    class CreateEnumBookingStatus < ActiveRecord::Migration
    
      def change
        create_enum :booking_status
      end
    
    end

You can now customize it.
    
    create_enum :booking_status, :name_limit => 50
    # The above is equivalent to saying
    # create_table :booking_statuses do |t|
    #   t.string :name, :limit => 50, :null => false
    # end

Now, when you create your Booking model, your migration should create a reference column for status id's and a foreign
key relationship to the booking\_statuses table.
    
    create_table :bookings do |t|
      t.integer :status_id

      t.timestamps
    end

    # Ideally, you would use a gem of some sort to handle foreign keys.
    execute "ALTER TABLE bookings ADD 'bookings_bookings_status_id_fk' FOREIGN KEY (status_id) REFERENCES booking_statuses (id);"

It's easier to use the `references` method if you intend to stick to the default naming convention for reference columns.

    create_table :bookings do |t|
      t.references :booking_status # Same as t.integer booking_status_id

      t.timestamps
    end
    
There are two methods added to Rails migrations:

##### create\_enum(enum\_name, options = {}, &block)

Creates a new enum table.  `enum_name` will be automatically pluralized.  The following options are supported:

- [:name\_column]  Specify the column name for name of the enum.  By default it's :name.  This can be a String or a Symbol
- [:description]  Set this to `true` to have a 'description' column generated.
- [:name\_limit]  Set this define the limit of the name column.
- [:desc\_limit]  Set this to define the limit of the description column
- [:active]  Set this to `true` to have a boolean 'active' column generated.  The 'active' column will have the options of NOT NULL and DEFAULT TRUE.
- [:timestamps]  Set this to `true` to have the timestamp columns (created\_at and updated\_at) generated

You can also pass in a block that takes a table object as an argument, like `create_table`.

Example:

    create_enum :booking_status

is the equivalent of

    create_table :booking_statuses do |t|
      t.string :name, :null => false
    end
    add_index :booking_statuses, [:name], :unique => true

In a more complex case:

    create_enum :booking_status, :name_column => :booking_name,
                                 :name_limit  => 50,
                                 :description => true,
                                 :desc_limit  => 100,
                                 :active      => true,
                                 :timestamps  => true

is the equivalent of
    
    create_table :booking_statuses do |t|
      t.string :booking_name, :limit => 50, :null => false
      t.string :description, :limit => 100
      t.boolean :active, :null => false, :default => true
      t.timestamps
    end
    add_index :booking_statuses, [:booking_name], :unique => true

You can also customize the creation process by using a block:

    create_enum :booking_status do |t|
      t.boolean :first_booking, :null => false
    end

is the equivalent of

    create_table :booking_statuses do |t|
      t.string :name, :null => false
      t.boolean :first_booking, :null => false
    end
    add_index :booking_statuses, [:name], :unique => true

Notice that a unique index is automatically created on the specified name column.

##### remove\_enum(enum\_name)

Drops the enum table.  `enum_name` will be automatically pluralized.

Example:

    remove_enum :booking_status
    
is the equivalent of
    
    drop_table :booking_statuses

### acts\_as\_enumerated

    class BookingStatus < ActiveRecord::Base
      acts_as_enumerated  :conditions        => 'optional_sql_conditions',
                          :order             => 'optional_sql_order_by',
                          :on_lookup_failure => :optional_class_method, #This also works: lambda{ |arg| some_custom_action }
                          :name_column       => 'optional_name_column'  #If required, may override the default name column
    end

With that, your BookingStatus class will have the following methods defined:

#### Class Methods

##### [](*args)

`BookingStatus[arg]` performs a lookup for the BookingStatus instance for the given arg.  The arg value can be a
'string' or a :symbol, in which case the lookup will be against the BookingStatus.name field.  Alternatively arg can be
a Fixnum, in which case the lookup will be against the BookingStatus.id field.  Since version 0.5.3, it returns the arg
if arg is an instance of the enum (in this case BookingStatus) as a convenience.

The `:on_lookup_failure` option specifies the name of a *class* method to invoke when the `[]` method is unable to
locate a BookingStatus record for arg.  The default is the built-in `:enforce_none` which returns nil. There are also
built-ins for `:enforce_strict` (raise and exception regardless of the type for arg), `:enforce_strict_literals` (raises
an exception if the arg is a Fixnum or Symbol), `:enforce_strict_ids` (raises and exception if the arg is a Fixnum) and
`:enforce_strict_symbols` (raises an exception if the arg is a Symbol).

The purpose of the `:on_lookup_failure` option is that a) under some circumstances a lookup failure is a Bad Thing and
action should be taken, therefore b) a fallback action should be easily configurable.  As of version 0.8.4, you can
also set `:on_lookup_failure` to a lambda that takes in a single argument (The arg that was passed to `[]`).

As of version 0.8.0, you can pass in multiple arguments to `[]`.  This returns a list of enums corresponding to the
passed in values.  Duplicates are filtered out.  For example `BookingStatus[arg1, arg2, arg3]` would be equivalent to
`[BookingStatus[arg1], BookingStatus[arg2], BookingStatus[arg3]]`.

##### all

`BookingStatus.all` returns an array of all BookingStatus records that match the `:conditions` specified in
`acts_as_enumerated`, in the order specified by `:order`.

##### active

`BookingStatus.active` returns an array of all BookingStatus records that are marked active.  See the `active?` instance
method.

##### inactive

`BookingStatus.inactive` returns an array of all BookingStatus records that are inactive.  See the `inactive?` instance
method.

##### names (since version 0.6.3)

`BookingStatus.names` will return all the names of the defined enums as an array of symbols.

##### update\_enumerations\_model (since version 0.8.1)

The preferred mechanism to update an enumerations model in migrations and similar.  Pass in a block to this method to
to perform any updates.

Example:

    BookingStatus.update_enumerations_model do
      BookingStatus.create :name        => 'Foo',
                           :description => 'Bar',
                           :active      => false
    end

#### Instance Methods

Each enumeration model gets the following instance methods.

##### ===(arg)

Behavior depends on the type of `arg`.

* If `arg` is `nil`, returns `false`.
* If `arg` is an instance of `Symbol`, `Fixnum` or `String`, returns the result of `BookingStatus[:foo] == BookingStatus[arg]`.
* If `arg` is an `Array`, returns `true` if any member of the array returns `true` for `===(arg)`, `false` otherwise.
* In all other cases, delegates to `===(arg)` of the superclass.

Examples:

    BookingStatus[:foo] === :foo #Returns true
    BookingStatus[:foo] === 'foo' #Returns true
    BookingStatus[:foo] === :bar #Returns false
    BookingStatus[:foo] === [:foo, :bar, :baz] #Returns true
    BookingStatus[:foo] === nil #Returns false

You should note that defining an `:on_lookup_failure` method that raises an exception will cause `===` to also raise an
exception for any lookup failure of `BookingStatus[arg]`.

`like?` is aliased to `===`

##### in?(*list)

Returns true if any element in the list returns true for `===(arg)`, false otherwise.

Example:

    BookingStatus[:foo].in? :foo, :bar, :baz #Returns true

##### name

Returns the 'name' of the enum, i.e. the value in the `:name_column` attribute of the enumeration model.

##### name\_sym

Returns the symbol representation of the name of the enum.  `BookingStatus[:foo].name_sym` returns :foo.

##### active?

Returns true if the instance is active, false otherwise.  If it has an attribute 'active',
returns the attribute cast to a boolean, otherwise returns true.  This method is used by the `active`
class method to select active enums.

##### inactive?

Returns true if the instance is inactive, false otherwise.  Default implementations returns `!active?`
This method is used by the `inactive` class method to select inactive enums.

#### Notes

`acts_as_enumerated` records are considered immutable. By default you cannot create/alter/destroy instances because they
are cached in memory.  Because of Rails' process-based model it is not safe to allow updating acts\_as\_enumerated
records as the caches will get out of sync.  Also, as of version 0.5.1, `to_s` is overriden to return the name of the
enum instance.

However, one instance where updating the models *should* be allowed is if you are using seeds.rb to seed initial values
into the database.

Using the above example you would do the following:

    BookingStatus.enumeration_model_updates_permitted = true
    ['pending', 'confirmed', 'canceled'].each do | status_name |
        BookingStatus.create( :name => status_name )
    end

Note that a `:presence` and `:uniqueness` validation is automatically defined on each model for the name column.

### has\_enumerated

First of all, note that you *could* specify the relationship to an `acts_as_enumerated` class using the `belongs_to`
association.  However, `has_enumerated` is preferable because you aren't really associated to the enumerated value, you
are *aggregating* it. As such, the `has_enumerated` macro behaves more like an aggregation than an association.

    class Booking < ActiveRecord::Base
      has_enumerated  :status, :class_name        => 'BookingStatus',
                               :foreign_key       => 'status_id',
                               :on_lookup_failure => :optional_instance_method,
                               :permit_empty_name => true,  #Setting this to true disables automatic conversion of empty strings to nil.  Default is false.
                               :default           => :unconfirmed,  #Default value of the attribute.
                               :create_scope      => false  #Setting this to false disables the automatic creation of the 'with_status' scope.
    end

By default, the foreign key is interpreted to be the name of your has\_enumerated field (in this case 'booking\_status')
plus '\_id'.  Since we chose to make the column name 'status\_id' for the sake of brevity, we must explicitly designate
it.  Additionally, the default value for `:class_name` is the camelized version of the name for your has\_enumerated
field. `:on_lookup_failure` is explained below.  `:permit_empty_name` is an optional flag to disable automatic
conversion of empty strings to nil.  It is typically desirable to have `booking.update_attributes(:status => '')`
assign status\_id to a nil rather than raise an Error, as you'll be often calling `update_attributes` with form data, but
the choice is yours.  Setting a `:default` option will generate an after\_initialize callback to set a default value on
the attribute unless a non-nil value has already been set.

With that, your Booking class will have the following methods defined:

#### status

Returns the BookingStatus with an id that matches the value in the Booking.status\_id.

#### status=(arg)

Sets the value for Booking.status\_id using the id of the BookingStatus instance passed as an argument.  As a
short-hand, you can also pass it the 'name' of a BookingStatus instance, either as a 'string' or :symbol, or pass in the
id directly.

example:

    mybooking.status = :confirmed
    
this is equivalent to:

    mybooking.status = 'confirmed'

or:

    mybooking.status = BookingStatus[:confirmed]

The `:on_lookup_failure` option in has\_enumerated is there because you may want to create an error handler for
situations where the argument passed to `status=(arg)` is invalid.  By default, an invalid value will cause an
ArgumentError to be raised.

Of course, this may not be optimal in your situation.  In this case you can do one of three things:

1) You can set it to 'validation\_error'.  In this case, the invalid value will be cached and returned on
subsequent lookups, but the model will fail validation.

2) Specify an *instance* method to be called in the case of a lookup failure. The method signature is as follows:

    your_lookup_handler(operation, name, name_foreign_key, acts_enumerated_class_name, lookup_value)

The 'operation' arg will be either `:read` or `:write`.  In the case of `:read` you are expected to return something or
raise an exception, while in the case of a `:write` you don't have to return anything.

Note that there's enough information in the method signature that you can specify one method to handle all lookup
failures for all has\_enumerated fields if you happen to have more than one defined in your model.

3) (Since version 0.8.5) Give it a lambda function.  In that case, the lambda needs to accept the ActiveRecord model as
its first argument, with the rest of the arguments being identical to the signature of the lookup handler instance
method.

    :on_lookup_failure => lambda{ |record, op, attr, fk, cl_name, value|
       # handle lookup failure
    }

NOTE: A `nil` is always considered to be a valid value for `status=(arg)` since it's assumed you're trying to null out
the foreign key.  The `:on_lookup_failure` will be bypassed.

#### with\_enumerated\_attribute scope

Unless the `:create_scope` option is set to `false`, a scope is automatically created that takes a list of enums as
arguments.  This allows us to say things like:

    Booking.with_status :confirmed, :received

Strings, symbols, ids, or enum instances are all valid arguments.  For example, the following would be valid, though not
recommended for obvious reasons.

    Booking.with_status 1, 'confirmed', BookingStatus[:rejected]

As of version 0.5.5, it also aliases a pluralized version of the scope, i.e. `:with_statuses`

#### exclude\_enumerated\_attribute scope

As of version 0.8.0, a scope for the inverse of `with_enumerated_attribute` is created, unless the `:create_scope`
option is set to `false`.  As a result, this allows us to do things like

    Booking.exclude_status :received

This will give us all the Bookings where the status is a value other than `BookingStatus[:received]`.

NOTE: This will NOT pick up instances of Booking where status is nil.

A pluralized version of the scope is also created, so `Booking.exclude_statuses :received, :confirmed` is valid.

### ActiveRecord::Base Extensions

The following methods are added to ActiveRecord::Base as class methods.

#### has\_enumerated?(attr)

Returns true if the given attr is an enumerated attributes, false otherwise.  `attr` can be a string or a symbol.  This
is a class method.

#### enumerated\_attributes

Returns an array of attributes which are enumerated.

### ActiveRecord::VirtualEnumerations

In many instances, your `acts_as_enumerated` classes will do nothing more than just act as enumerated.

In that case there isn't much point cluttering up your models directory with those class files. You can use
ActiveRecord::VirtualEnumerations to reduce that clutter.

Copy virtual\_enumerations\_sample.rb to Rails.root/config/initializers/virtual\_enumerations.rb and configure it
accordingly.

See virtual\_enumerations\_sample.rb in the examples directory of this gem for a full description.

### Testing (Since version 0.6.0)

A pair of custom RSpec matchers are included to streamline testing of enums and enumerated attributes.

#### act\_as\_enumerated

This is used to test that a model acts as enumerated.  Example:

    describe BookingStatus do
      it { should act_as_enumerated }
    end

This also works:

    describe BookingStatus do
      it "should act as enumerated" do
        BookingStatus.should act_as_enumerated
      end
    end

You can use the `with_items` chained matcher to test that each enum is properly seeded:

    describe BookingStatus do
      it {
        should act_as_enumerated.with_items(:confirmed, :received, :rejected)
      }
    end

You can also pass in hashes if you want to be thorough and test out all the attributes of each enum.  If
you do this, you must pass in the `:name` attribute in each hash

    describe BookingStatus do
      it {
        should act_as_enumerated.with_items({ :name => 'confirmed', :description => "Processed and confirmed" },
                                            { :name => 'received', :description => "Pending confirmation" },
                                            { :name => 'rejected', :description => "Rejected due to internal rules" })
      }
    end

#### have\_enumerated

This is used to test that a model has enumerated the given attribute:

    describe Booking do
      it { should have_enumerated(:status) }
    end

This is also valid:

    describe Booking do
      it "Should have enumerated the status attribute" do
        Booking.should have_enumerated(:status)
      end
    end

## How to run tests

Go to the 'dummy' project:
    
    cd ./spec/dummy

Run migrations for test environment:

    RAILS_ENV=test rake db:migrate

You may need to use `bundle exec` if you have gem conflicts:

    RAILS_ENV=test bundle exec rake db:migrate

Go back to gem root directory:

    cd ../../

And finally run tests:

    rake spec

## Copyrights and License

* Initial Version Copyright (c) 2005 Trevor Squires
* Rails 3 Updates Copyright (c) 2010 Pivotal Labs
* Initial Test Suite Copyright (c) 2011 Sergey Potapov
* Subsequent Updates Copyright (c) 2011 Arthur Shagall

Released under the MIT License.  See the LICENSE file for more details.

## Contributing

Contributions are welcome.  However, before issuing a pull request, please make sure of the following:

* All specs are passing.
* Any new features have test coverage.
* Anything that breaks backward compatibility has a very good reason for doing so.
