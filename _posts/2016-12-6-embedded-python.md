---
layout: post
title: Embedding python in C++ with boost::python
---

We have a low-latency C++ trading framework which allows us to deploy various trading strategies relatively easily.

Most of our traders, however, don't know C++, which means they are often waiting on IT resource to develop and deploy a new strategy.

A lot of them do know python though, so I wanted to find a way to allow them to write the core logic of their strategies in python, whilst still leveraging our low-latency trading infrastructure

Here is my *embedding python in C++* proof-of-concept.

There is nothing new here, but I found existing documentation and tutorials somewhat lacking, hence this blog post.

# Step 1: Define the C++ types which represents the trading framework

### Side

*An enum to represent buying or selling an order*

```cpp
enum Side
{
    BUY,
    SELL
};
```

### Order

*A struct with the details of a single order*

```cpp
struct Order
{
    std::string symbol;
    Side        side;
    int         size;
    double      price;
    int         order_id;
};
```

### StrategyServer

*A class which provides the trading functionality, such as sending an order*

```cpp
class StrategyServer
{
public:
    void sendOrder(StrategyInstance&, const std::string& symbol, Side, int size, double price);
private:
    int _next_order_id = 0;
};
```

### StrategyInstance

*A pure virtual base class which represents a single strategy*

- Subclasses must override `eval()`,  the entry-point into the strategy. This is where the strategy's logic is evaluated and orders potentially sent to market.

- Subclasses must also override `onOrder(const Order&)` which is the callback for the resulting order sent to market

```cpp
class StrategyInstance
{
public:
    StrategyInstance(StrategyServer&);
    virtual ~StrategyInstance() = default;

    // evaluates the strategy's logic
    virtual void eval() = 0;
    
    // callback for an acknowledged order
    virtual void onOrder(const Order&) = 0;

    // asks the StrategyServer to send an order
    void sendOrder(const std::string& symbol, Side, int size, double price); 

private:
    StrategyServer& _server;
};
```

# Step 2: Create a python wrapper for our virtual StrategyInstance

Use **`boost::python::wrapper`** to allow a python class to subclass StrategyInstance

The virtual overrides look up the python class's implementation and call those

```cpp
class PyStrategyInstance final
    : public StrategyInstance
    , public bp::wrapper<StrategyInstance>
{
    using StrategyInstance::StrategyInstance;

    void eval() override
    {
        // call through to the python class's `eval` method
        get_override("eval")();
    }

    void onOrder(const Order& order) override
    {
        // call through to the python class's `on_order` method
        get_override("on_order")(order);
    }
};
```

# Step 3: Expose these types to python

We use **`BOOST_PYTHON_MODULE`** to define a python module containing all these types

```cpp
BOOST_PYTHON_MODULE(StrategyFramework)
{
    bp::enum_<Side>("Side")
        .value("BUY", BUY)
        .value("SELL", SELL)
        ;

    bp::class_<Order>("Order")
        .def_readonly("symbol",   &Order::symbol)
        .def_readonly("side",     &Order::side)
        .def_readwrite("size",    &Order::size)
        .def_readwrite("price",   &Order::price)
        .def_readonly("order_id", &Order::order_id)
        ;

    bp::class_<StrategyServer>("StrategyServer")
        ;

    bp::class_<PyStrategyInstance, boost::noncopyable>("StrategyInstance", bp::init<StrategyServer&>())
        .def("send_order", &StrategyInstance::sendOrder)
        ;
}
```

# Step 4: define a function which imports a python file as a module

This is taken from the python wiki on `boost::python`, from the tip on [loading a module by path](https://wiki.python.org/moin/boost.python/EmbeddingPython#Loading_a_module_by_full_or_relative_path).

What is does is allow us to specify a python file and load it as if we called **`import module`**

```cpp
bp::object import(const std::string& module, const std::string& path, bp::object& globals)
{
    bp::dict locals;
    locals["module_name"] = module;
    locals["path"]        = path;

    bp::exec("import imp\n"
             "new_module = imp.load_module(module_name, open(path), path, ('py', 'U', imp.PY_SOURCE))\n",
             globals,
             locals);
    return locals["new_module"];
}
```

# Step 5: load a python file containing our strategy and execute it

This is where we pull it all together.

We initialise the python runtime, register the python module with our C++ code in it, import the python file which contains our strategy, and then run it

```cpp
int main()
try
{
    Py_Initialize();
    
    // register the python module we created, so our script can import it
    PyImport_AppendInittab("StrategyFramework", &initStrategyFramework);

    StrategyServer server;

    // import the __main__ module and obtain the globals dict
    bp::object main     = bp::import("__main__");
    bp::object globals  = main.attr("__dict__");
    
    // import our strategy.py file
    bp::object module   = import("strategy", "strategy.py", globals);
    
    // obtain the strategy class and instantiate one
    bp::object Strategy = module.attr("Strategy");
    bp::object strategy = Strategy(server);

    // eval the strategy
    strategy.attr("eval")();

    return 0;
}
catch(const bp::error_already_set&)
{
    std::cerr << ">>> Error! Uncaught exception:\n";
    PyErr_Print();
    return 1;
}
```

# Step 6: define a python strategy

This is the sample strategy written in python.

As a proof of concept, all it does is send an order when the strategy is evaluated.

We print the resulting order we receive in our callback

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

# Step 7: run it!

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
#include <boost/python.hpp>

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

// forward declaration
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
    StrategyInstance(StrategyServer&);
    virtual ~StrategyInstance() = default;

    virtual void eval() = 0;
    virtual void onOrder(const Order&) = 0;

    void sendOrder(const std::string& symbol, Side, int size, double price);

private:
    StrategyServer& _server;
};

///////////////////////////////////

// implementation

void StrategyServer::sendOrder(StrategyInstance& instance, const std::string& symbol, Side side, int size, double price)
{
    // simulate sending an order, receiving an acknowledgement and calling back to the strategy instance

    std::cout << "sending order to market\n";

    Order order { symbol, side, size, price, ++_next_order_id };
    instance.onOrder(order);
}

///////////////////////////////////

StrategyInstance::StrategyInstance(StrategyServer& server)
    : _server(server)
{}

void StrategyInstance::sendOrder(const std::string& symbol, Side side, int size, double price)
{
    _server.sendOrder(*this, symbol, side, size, price);
}

////////////////////////////////////

// export to python 

namespace bp = boost::python;

class PyStrategyInstance final
    : public StrategyInstance
    , public bp::wrapper<StrategyInstance>
{
    using StrategyInstance::StrategyInstance;

    void eval() override
    {
        get_override("eval")();
    }

    void onOrder(const Order& order) override
    {
        get_override("on_order")(order);
    }
};

BOOST_PYTHON_MODULE(StrategyFramework)
{
    bp::enum_<Side>("Side")
        .value("BUY", BUY)
        .value("SELL", SELL)
        ;

    bp::class_<Order>("Order")
        .def_readonly("symbol",   &Order::symbol)
        .def_readonly("side",     &Order::side)
        .def_readwrite("size",    &Order::size)
        .def_readwrite("price",   &Order::price)
        .def_readonly("order_id", &Order::order_id)
        ;

    bp::class_<StrategyServer>("StrategyServer")
        ;

    bp::class_<PyStrategyInstance, boost::noncopyable>("StrategyInstance", bp::init<StrategyServer&>())
        .def("send_order", &StrategyInstance::sendOrder)
        ;
}

bp::object import(const std::string& module, const std::string& path, bp::object& globals)
{
    bp::dict locals;
    locals["module_name"] = module;
    locals["path"]        = path;

    bp::exec("import imp\n"
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
    PyImport_AppendInittab("StrategyFramework", &initStrategyFramework);

    StrategyServer server;

    bp::object main     = bp::import("__main__");
    bp::object globals  = main.attr("__dict__");
    bp::object module   = import("strategy", "strategy.py", globals);
    bp::object Strategy = module.attr("Strategy");
    bp::object strategy = Strategy(server);

    strategy.attr("eval")();

    return 0;
}
catch(const bp::error_already_set&)
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
