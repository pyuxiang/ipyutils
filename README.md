# ipyutils

> The result of my poor naming skills suggests this library is related to IPython. It is not.

A dependency-free utility script that facilitates class-backed cross-platform inter-python communication, packaged into a simple module.

This is an RPC implementation, and not a message broker implementation - some salient aspects of this library includes:

* Fully synchronous messaging (as opposed to fault-resilient asynchronous distributed N-to-N messaging)
* Primitives are Python class methods and arguments (rather than lower-level data)
* Data encoding is natively handled by pickling (so much less efficient compared to schema-defined data serialization)

## Installation

Only requires Python 3.8+. There are no other external dependencies.

```bash
pip install ipyutils
```

## Why?

Use case was borne from the need to communicate with Python scripts sitting across on a different computer within the same network (e.g. Python wrappers for controlling shared devices in an experimental physics lab), without resorting to repeated SSH connections and/or device initialization.

For example:

* Computer A has a locally connected device over wire, controlled using a `Powermeter` class.
* Computer B is the main workstation running a control script that needs to access this device.

On the computer A sits a script that functionally looks like:

```python
from ipyutils import Server
from MyModule import Powermeter

pm = Powermeter(device=...)  # has methods like 'read_power(averaging=100)'
# perhaps some initialization here
Server(pm).run()  # exposed on LAN address 192.168.1.2
```

while on computer B, the control script can control the device on computer A, as if it were also locally connected:

```python
from ipyutils import Client

pm = Client(address="192.168.1.2")
# optionally do even more initialization here
power = pm.read_power(averaging=100)  # this command is run on computer A, with results piped back here
```

That's all. Because any low-level device messages are processed locally on the connected computer, this minimizes query latencies with the device itself (e.g. receiving 100 power readings over LAN, instead of just over-the-wire).

The experience becomes even more native when the Python class itself can be referenced on the client, to allow transparent access to the properties of the instance as well. Custom functions can also be registered. For an explanation of all the nice features of this library, see the [detailed user guide](docs/ipcutil.md).

### Summary of implementation

The bulk of the work is handled by `multiprocessing.connection`, with a thin control plane sitting above it to handle message receipt acknowledgements. Data is serialized by pickling. Peer discovery is via hard-coded TCP address and ports, especially since shared devices aren't expected to move between computers/networks.

### Good to use when:

* Most of your scripts are already in Python, and hence would already have compatible code.
* Runtime bottlenecks come from device IO latency or downstream data processing
  * As opposed to network bandwidth or latency, or data serialization.
* Internal network is reasonably trusted and your network is friendly to listening TCP ports.
* Device usage/access is within your control, or shared between people you can reason with.
  * In a multi-tenant environment, everyone needs to agree on releasing connections between usage, because access is limited only to a single client at any time.
* Data privacy is not required, since data is transferred unencrypted.

### Other perks include:

* Time-sharing of the same device between (respectful) clients.
* Request blocks indefinitely until server comes back online (so no need to have timeout fallbacks).
* No need to define complex data structures for packing and unpacking, through the use of pickling.
* Small codebase => very auditable 😄

## Alternative libraries

On hindsight, this style of pure-Python RPC is obviously very useful so other people would have done similar things. A list of similar RPC libraries I found when writing this that you may consider as alternatives, and are likely to be more well-tested:

* [`Pyro5`](https://pyro5.readthedocs.io/en/latest/index.html): Probably the closest equivalent implementation to this library, packed with lots of goodies like custom serializers, nameservers. However, there is [no support for `numpy` arrays](https://pyro5.readthedocs.io/en/latest/tipstricks.html#pyro-and-numpy), which I'd think is bread-and-butter for scientific computing. Supports Python 3.9+.
* [`RPyC`](https://rpyc.readthedocs.io/en/latest/): Implements RPC for the remote Python interpreter itself, how cool is that! Probably needs some massaging to fit this use case / syntactic sugar.
* [`xmlrpc`](https://docs.python.org/3/library/xmlrpc.html): Built-in library using XML-over-HTTP if only standard libraries are acceptable. Requires individual function registration.

## Other thoughts

The library has been used extensively in "production" (really just within my experimental research lab), so correctness has been manually verified - "dogfooding" so to speak. When in doubt, reading the source is highly recommended, which is basically a single simple <1k lines script.

This library was broken off from another LGPLv2.1 library of mine, so the old commits have been retained for provenance and license adherence.

No AI was used, as evidenced by my slow incremental upgrades. That being said, this library may likely be scrapped for AI training at some point, which blatantly disrespects all forms of copyright. I can only hope the copyleft wording of the LGPLv2.1 license is at least retained in spirit.
