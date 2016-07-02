# Gaw
 
*"gaw" means "glue" in Thai*

**Gaw** is a small library that helps you developing microservices over simple TCP socket with ease.

This is how it works!

**On the server side** (say, `services.py`)

```
from gaw import entrypoint

class MathService(object):
    name = 'math_service'
    
    def __init__(self, hello_msg):
    	self.msg = hello_msg

    @entrypoint # expose this method to the rest of the world
    def plus(self, a, b):
		return '{}:{}'.format(self.msg, a + b)

    @entrypoint
    def multiply(self, a, b):
		return '{}:{}'.format(self.msg, a * b)
```

You can start the server using `GawServer` like:

```
from services import MathService
from gaw import GawServer

GawServer('127.0.0.1', 5555).add(MathService, 'hello!').run() # runs forever
```

Alternatively, using a command-line interface

```
$ gaw services --kwargs="hello_msg='hello!'" # runs all services in the module 'services'
# or
$ gaw services --service=math_service --kwargs="hello_msg='hello!'"
```

Anyways, usually, we don't really need the use of parameters like the above.

**On the client side**

```
from gaw import GawClient

client = GawClient('127.0.0.1', 5555)
rpc = client.math_service
print(rpc.plus(10, 20)) # Hello!: 30
print(rpc.multiply(10, 20)) # Hello!: 200
```

In some scenarios, you might need this

```
from gaw import GawServer, entrypoint
from somewhere import MathEngine

class MathService(object):
    name = 'math_service'
    math_engine = MathEngine()

    def __init__(self, hello_message):
        self.hello = hello_message

    @entrypoint # expose this method to the rest of the world
    def plus(self, a, b):
        return '{}: {}'.format(self.hello, self.math_engine.plus(a, b))

    @entrypoint
    def multiply(self, a, b):
        return '{}: {}'.format(self.hello, self.math_engine.multiply(a, b)))


service = GawServer('127.0.0.1', 5555)
service.add(MathService, hello_message='Hello!')
service.run() # runs forever
```

In the example above, you can guarantee that there should be only one MathEngine initiated.

**Gaw** is heavily influenced by **Nameko**, another python microservice framework.

*Note: it supports python 3.4, and tested with python 2.7.9*

## Installation

```
pip install gaw
```

## Request Life Cycle

1. **Gaw Client** makes a connection and sends a request packet to a **Gaw Server**.
2. **Gaw Server**, knowing all the entrypoints, **Gaw Server** *inititates* an instance of a designated class.
3. **Gaw Server** invokes the requested method.
4. **Gaw Server** sends back the results to the calling **Gaw Client**.


## Topology

In the package, there are other two libraries that **Gaw** makes use of:

1. **Postoffice** - serves as a low-level TCP socket communicator.
2. **Json Web Server** - this's kinda like a http server for the mere socket world.

## Suggestions

You may have your own "ways" of doing microservice, but I tend to use this pattern.

config.py

```
MICROSERVICES = dict(
    AService=dict(
        ip='<host1>',
        port=<port1>),
    BService=dict(
        ip='<host2>',
        port=<port2>),
    CService=dict(
        ip='<host3>',
        port=<port3>),
)
```

utils.py

```
from config import MICROSERVICES
from gaw import GawServer, GawClient

def pluck(d, *args):
    assert isinstance(d, dict)
    return (d[arg] for arg in args)

def rpc_of(service_name):
    ip, port = pluck(MICROSERVICES[service_name], 'ip', 'port')
    return getattr(GawClient(ip=ip, port=port), service_name) # equals to GawClient(..).<service_name>
    
def ip_port_of(service_name):
    ip, port = pluck(MICROSERVICES[service_name], 'ip', 'port')
    return ip, port

def run_service(service_class):
    service_name = service_class.name
    ip, port = ip_port_of(service_name)
    GawServer(ip, port).add(service_class).run()
```

run_server.py

```
from utils import run_service
from <service> import AService

run_service(AService)
```

client.py

```
from utils import rpc_of

AService = rpc_of('AService')
AService.<method_name>(...)
```
