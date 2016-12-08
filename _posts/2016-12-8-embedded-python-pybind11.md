---
layout: post
title: Embedding python in C++ with pybind11
---

I recently wrote a [blog post]({% post_url 2016-12-6-embedded-python %}) about embedding python in C++ with boost::python.

In the [reddit discussion](https://www.reddit.com/r/cpp/comments/5gwor7/embedding_python_in_c_with_boostpython/), several people made reference to pybind11.

The last time I looked at pybind11 it didn't support embedding, hence me not considering it for the proof of concept.

It was pointed out to me, however, that it is now possible to embed python with pybind11, so I decided to try it out.

I make use of the same C++ classes as in the previous example, with a few minor changes. For an overview of the class hierarchy, please see the [previous blog post]({% post_url 2016-12-6-embedded-python %}).

# Changes required in the C++ hierarchy

The most notable change was my `StrateyInstance` class.

In the boost::python example the `StrategyServer` was taken by reference. 

I found that my python springboard class (`PyStrategyInstance`) was using pass-by-value semantics, leaving me with a reference to garbage. I was unable to figure out how to pass-by-reference. If anyone knows how to do this, please let me know! 

As a result I had to change the `StrategyInstance` constructor to take a `StrategyServer` pointer.

```cpp
class StrategyInstance
{
public:
    StrategyInstance(StrategyServer*); // StrategyServer now passed as a pointer
    virtual ~StrategyInstance() = default;

    virtual void eval() = 0;
    virtual void onOrder(const Order&) = 0;

    void sendOrder(const std::string& symbol, Side, int size, double price);

private:
    StrategyServer& _server;
};
```

# Python wrapper for our virtual StrategyInstance

Here the real meat of the changes starts.

As in boost::python, we need to make a springboard class which overrides the pure virtual base class functions, and calls through to the python overload.

The difference, however, is that instead of inheriting from `bp::wrapper`, we use the `PYBIND11_OVERLOAD_XXX` macros to call through to python

```cpp
class PyStrategyInstance final
    : public StrategyInstance
{
    using StrategyInstance::StrategyInstance;

    void eval() override
    {
        PYBIND11_OVERLOAD_PURE(
            void,              // return type
            StrategyInstance,  // super class
            eval               // function name
            );
    }

    void onOrder(const Order& order) override
    {
        PYBIND11_OVERLOAD_PURE_NAME(
            void,              // return type
            StrategyInstance,  // super class
            "on_order",        // python function name
            onOrder,           // function name
            order              // args
            );
    }
};
```

# Exposing the C++ types to python

Creating a module (or *plugin* in pybind11 parlance) is largely similar to boost::python.

A major difference is the requirement to create a `module` inside the plugin, and pass that to all the types defined in the plugin, and then we need to return the module's `PyObject*` at the end of our plugin definition.

We use **`PYBIND11_PLUGIN`** to define our python plugin

```cpp
PYBIND11_PLUGIN(StrategyFramework)
{
    // we need to create a module
    py::module m("StrategyFramework", "Example strategy framework");

    py::enum_<Side>(m, "Side")
        .value("BUY", BUY)
        .value("SELL", SELL)
        ;

    py::class_<Order>(m, "Order")
        .def_readonly ("symbol",   &Order::symbol)
        .def_readonly ("side",     &Order::side)
        .def_readwrite("size",     &Order::size)
        .def_readwrite("price",    &Order::price)
        .def_readonly ("order_id", &Order::order_id)
        ;

    py::class_<StrategyServer>(m, "StrategyServer")
        ;

    py::class_<StrategyInstance, PyStrategyInstance>(m, "StrategyInstance")
        .def(py::init<StrategyServer*>())
        .def("send_order", &StrategyInstance::sendOrder)
        ;

    // return the module's PyObject*
    return m.ptr();
}
```

# Defining a function which imports a python file as a module

I used the same function I found in the python wiki on `boost::python`, from the tip on [loading a module by path](https://wiki.python.org/moin/boost.python/EmbeddingPython#Loading_a_module_by_full_or_relative_path).

What it does is allow us to specify a python file and load it as if we called **`import module`**

A notable difference between boost::python and pybind11 is that boost::python `dicts` can be passed `std::string` directly in the assignment of a value. In pybind11 you have to use `py::cast`

You also gave to tell `py::eval` that you are passing it multiple statements (`py::eval<py::eval_statements>(...)`)

```cpp
py::object import(const std::string& module, const std::string& path, py::object& globals)
{
    py::dict locals;
    locals["module_name"] = py::cast(module); // have to cast the std::string first
    locals["path"]        = py::cast(path);

    py::eval<py::eval_statements>(            // tell eval we're passing multiple statements
        "import imp\n"
        "new_module = imp.load_module(module_name, open(path), path, ('py', 'U', imp.PY_SOURCE))\n",
        globals,
        locals);

    return locals["new_module"];
}
```

# Initialising our plugin/module 

As in the boost::python example, we need to import our module.

In boost::python the `BOOST_PYTHON_MODULE` macro defines a function `void initModuleName()` (where ModuleName is the name of the module passed to the macro). We are able to use the `libpython` C api to initialise our module:

**boost::python way:**

```cpp
// this is how we register our python module when using boost::python
PyImport_AppendInittab("StrategyFramework", &initStrategyFramework);
```

**pybind11 way:**

In pybind11 the usage is slightly different. pybind11's `PYBIND11_PLUGIN` macro defines a function `PyObject* pybind11_init()` which we can call to initialise our module.

The slightly unfortunate thing with the way pybind11 defines this function is that the name (`pybind11_init`) is the same for all plugins. This means that if you want to define and/or initialise more than one plugin, you need to define them in separate namespaces, otherwise you'll violate ODR.

```cpp
namespace plugin1 
{
    PYBIND11_PLUGIN(Foo)
    {
        static PyObject* pybind11_init(); // defined for us by the macro
    }
}
namespace plugin2
{
    PYBIND11_PLUGIN(Bar)
    {
        static PyObject* pybind11_init(); // defined for us by the macro
    }
}

// initialise both plugins
plugin1::pybind11_init();
plugin2::pybind11_init();
```

Since I'm only using one plugin I don't need to do this in my code, so I haven't bothered putting my plugin into a separate namespace.

## Pulling it all together

Here we initialise the python runtime, initialise the python module with our C++ code in it, import the python file which contains our strategy, and then run it

```cpp
int main()
try
{
    Py_Initialize();
    pybind11_init();

    StrategyServer server;

    py::object main     = py::module::import("__main__");
    py::object globals  = main.attr("__dict__");
    py::object module   = import("strategy", "strategy.py", globals);
    py::object Strategy = module.attr("Strategy");
    py::object strategy = Strategy(&server); // have to pass server as a pointer now

    strategy.attr("eval")();

    return 0;
}
catch(const py::error_already_set&)
{
    std::cerr << ">>> Error! Uncaught exception:\n";
    PyErr_Print();
    return 1;
}
```

Note here that, as in boost::python, I don't call `Py_Finalize()`. In both frameworks, a call to `Py_Finalise()` results in a segmentation fault.

# Define a python strategy

The sample strategy written in python requires no change.

```cpp
from StrategyFramework import *

class Strategy(StrategyInstance):
    def __init__(self, server):
        StrategyInstance.__init__(self, server)

    def eval(self):
        print "strategy.eval"
        self.send_order("GOOG", Side.BUY, 100, 759.11)

    def on_order(self, o):
        print "order for {} {} {}@{} has order_id={}".format(
            o.symbol, "buy" if o.side == Side.BUY else "sell", o.size, o.price, o.order_id)
```

# Run it!

```
$ ./strategy 
strategy.eval
sending order to market
order for GOOG buy 100@759.11 has order_id=1
```

---

# Complete code listing:

Here is the complete code listing which you can use to test out the code yourself

## `main.cpp`:

```cpp
#include <iostream>
#include <pybind11/pybind11.h>
#include <pybind11/eval.h>

enum Side
{
    BUY,
    SELL
};

struct Order
{
    std::string symbol;
    Side        side;
    int         size;
    double      price;
    int         order_id;
};

class StrategyInstance;

class StrategyServer
{
public:
    void sendOrder(StrategyInstance&, const std::string& symbol, Side, int size, double price);
private:
    int _next_order_id = 0;
};

class StrategyInstance
{
public:
    StrategyInstance(StrategyServer*);
    virtual ~StrategyInstance() = default;

    virtual void eval() = 0;
    virtual void onOrder(const Order&) = 0;

    void sendOrder(const std::string& symbol, Side, int size, double price);

private:
    StrategyServer& _server;
};

///////////////////////////////////

void StrategyServer::sendOrder(StrategyInstance& instance, const std::string& symbol, Side side, int size, double price)
{
    // simulate sending an order, receiving an acknowledgement and calling back to the strategy instance

    std::cout << "sending order to market\n";

    Order order { symbol, side, size, price, ++_next_order_id };
    instance.onOrder(order);
}

///////////////////////////////////

StrategyInstance::StrategyInstance(StrategyServer* server)
    : _server(*server)
{
}

void StrategyInstance::sendOrder(const std::string& symbol, Side side, int size, double price)
{
    _server.sendOrder(*this, symbol, side, size, price);
}

////////////////////////////////////

namespace py = pybind11;

class PyStrategyInstance final
    : public StrategyInstance
{
    using StrategyInstance::StrategyInstance;

    void eval() override
    {
        PYBIND11_OVERLOAD_PURE(
            void,              // return type
            StrategyInstance,  // super class
            eval               // function name
            );
    }

    void onOrder(const Order& order) override
    {
        PYBIND11_OVERLOAD_PURE_NAME(
            void,              // return type
            StrategyInstance,  // super class
            "on_order",        // python function name
            onOrder,           // function name
            order              // args
            );
    }
};

PYBIND11_PLUGIN(StrategyFramework)
{
    py::module m("StrategyFramework", "Example strategy framework");

    py::enum_<Side>(m, "Side")
        .value("BUY", BUY)
        .value("SELL", SELL)
        ;

    py::class_<Order>(m, "Order")
        .def_readonly ("symbol",   &Order::symbol)
        .def_readonly ("side",     &Order::side)
        .def_readwrite("size",     &Order::size)
        .def_readwrite("price",    &Order::price)
        .def_readonly ("order_id", &Order::order_id)
        ;

    py::class_<StrategyServer>(m, "StrategyServer")
        ;

    py::class_<StrategyInstance, PyStrategyInstance>(m, "StrategyInstance")
        .def(py::init<StrategyServer*>())
        .def("send_order", &StrategyInstance::sendOrder)
        ;

    return m.ptr();
}

py::object import(const std::string& module, const std::string& path, py::object& globals)
{
    py::dict locals;
    locals["module_name"] = py::cast(module);
    locals["path"]        = py::cast(path);

    py::eval<py::eval_statements>(
        "import imp\n"
        "new_module = imp.load_module(module_name, open(path), path, ('py', 'U', imp.PY_SOURCE))\n",
        globals,
        locals);

    return locals["new_module"];
}

//////////////////////////////////

int main()
try
{
    Py_Initialize();
    pybind11_init();

    StrategyServer server;

    py::object main     = py::module::import("__main__");
    py::object globals  = main.attr("__dict__");
    py::object module   = import("strategy", "strategy.py", globals);
    py::object Strategy = module.attr("Strategy");
    py::object strategy = Strategy(&server);

    strategy.attr("eval")();
    return 0;
}
catch(const py::error_already_set&)
{
    std::cerr << ">>> Error! Uncaught exception:\n";
    PyErr_Print();
    return 1;
}
```

## `strategy.py`:

```cpp
from StrategyFramework import *

class Strategy(StrategyInstance):
    def __init__(self, server):
        StrategyInstance.__init__(self, server)

    def eval(self):
        print "strategy.eval"
        self.send_order("GOOG", Side.BUY, 100, 759.11)

    def on_order(self, o):
        print "order for {} {} {}@{} has order_id={}".format(
            o.symbol, "buy" if o.side == Side.BUY else "sell", o.size, o.price, o.order_id)

```
