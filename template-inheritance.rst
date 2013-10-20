
Templates and Virtual Functions Don't Play Nice
===============================================

Well, after spending a nice, long while on a templated design for a recent
project, I ran into a big no-go based on my lack of understanding of the way C++
templates and virtual functions work. It makes sense once you think about it,
but it was not at all obvious to me (hence my wasting multiple days implementing
something that wouldn't work). 

tl;dr: Virtual functions use vtables, which is a run-time method
binding. Template definitions are generate new functions/classes at
compile-time. When combined with dynamic loading, templates make it difficult to
determine the size of the vtable.

Introduction
------------

Cyclus uses a general pattern for implementing agents in a simulation. There is
a base agent-type class (Model) that defines the minimal interface for all
possible agents. Specializations are then defined as derived classes (e.g.,
FacilityModel, RegionModel, etc.) which extend and/or overload the basic Model
interface. Finally, externally-developed modules further specialize this
interface. Take for example the following:

.. code-block:: c++

  // base agent-type class
  class Base {
   public:
    // an optional, extendible part of the interface
    virtual std::string print() { return "base class";}

    // the rest of the interface...
  };   

  // specialized class
  class Derived : public Base {
   public:
    // an optional, extendible part of the interface
    virtual std::string print() { return Base::print() + " and derived class";}

    // a required-to-implement part of the interface
    virtual std::vector<Resource::Ptr> requests() = 0;

    // the rest of the interface...
  };
   
  // module class
  class Module : public Derived {
   public:
    // an optional, extendible part of the interface
    virtual std::string print() { return Derived::print() + " and module class";}

    // a required-to-implement part of the interface
    virtual std::vector<Resource::Ptr> requests() {
      Resource::Ptr pr = Resource::Create();
      std::vector<Resource::Ptr> vr;
      vr.push_back(pr);
      return vr;
    };

    // the rest of the interface...
  };

This pattern relies on downstream classes to extend the interface or provide
implementations of required functions for which there is no obvious default
implementation. A key (and perhaps obvious) observation, though, is that the
types of both the arguments and return values must be known at the core-level
(i.e., within the Base and/or Derived classes). 

While this pattern allows for flexibility by supporting dynamically-loadable
modules, it has inherent (again, obvious) stiffness in that the core must define
the minimal API. An example of this stiffness follows.

Cyclus encapsulates the notion of resources in a class of the same name, and
conceptually a resource has a quantity and a quality, defining some
characteristic of the resource. While the quantity can be represented across
resource types as a ``double``, the quality type is dependent on the resource
type, i.e.,

.. code-block:: c++

  // resource class
  class Resource {
   public:
    double qty;

    // the rest of the interface...
  };   

  // impl for some "generic resrouce"
  class GenericResource : public Resource {
   public:
    std::string quality;

    // the rest of the interface...
  };

  // impl for a "material"
  class Material : public Resource {
   public:
    IsoVector quality;    

    // the rest of the interface...
  };

Resources are generally passed around the Cyclus core via pointers, allowing the
core to treat all derived types collectively. In some instances, though,
operations are desired to be performed on the quality member (comparison, for
instance). As it turns out, there are basically two approaches to dealing with
this situation:

#. maintain derived-type information in the core
#. dynamically cast resource pointers to determine the resource type and act
   accordingly

Problem Statement
-----------------

In order to provide type-specific introspection of resources, I initially
attempted to use a template-based design. I was able to construct all the
required datastructures, templating them each on the type of resource being used
or compared. I ran into problems, however, when trying to integrate these
datastructures into the agent interface.

For illustration purposes, imagine that we have the following resource
datastructure:

.. code-block:: c++

  // request for a resource
  template<class T>
  class Request {
   public:
    T& target;

    // the rest of the interface...
  };   

In order to conform to the Cyclus agent pattern, I'd like to add something like:

.. code-block:: c++

   template<class T>
   virtual std::set< Request<T> > SendRequests();

If this were possible, it would allow module developers to implement their own
SendRequests function as appropriate. Unfortunately, this runs afoul of the way
templates and virtual functions are implemented in C++.

