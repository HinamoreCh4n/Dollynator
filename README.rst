*******
Dollynator
*******

|jenkins_build|

*A self-replicating autonomous Tribler exit-node.*

**Dollynator** (formerly PlebNet) is an Internet-deployed Darwinian reinforcement learning system based on self-replication. Also referred to as a *botnet for good*, it consists of many generations of autonomous entities living on VPS instances with VPN installed, running Tribler_ exit-nodes, and routing torrent traffic in our Tor-like network.

While providing privacy and anonymity for regular Tribler users, it is earning reputation in form of MB tokens stored on Trustchain, which are in turn put on sale for Bitcoin on a fully decentralized Tribler marketplace. Once the bot earns enough Bitcoin, it buys a new VPS instance using Cloudomate_, and finally self-replicates.

The name Dollynator pays tribute to Dolly the sheep (the first cloned mammal) and the artificial intelligence of Terminator. It might also remotely resemble Skynet, a self-aware network that went out of control.


Bootstrapping
=============

The first running node needs to be installed manually. One of the options is to buy a VPS using Cloudomate, and install Dollynator from a local system using the ``plebnet/clone/create-child.sh`` script.

::

   Usage: ./create-child.sh [options]
      -h --help              Shows this help message
      -i --ip                Ip address of the server to run install on
      -p --password          Root password of the server
      -t --testnet           Install agent in testnet mode (default 0)
      -e --exitnode          Run as exitnode for tribler
      -conf --config         (optional) VPN configuration file (.ovpn)
                             Requires the destination config name.
                             Example: -conf source_config.ovpn dest_config.ovpn
      -cred --credentials    (optional) VPN credentials file (.conf)
                             Requires the destination credentials name.
                             Example -cred source_credentials.conf dest_credentials.conf
      -b --branch            (optional) Branch of code to install from (default master)


Example:

.. code-block:: console

    ./create-child.sh -i <ip> -p <password> -e -b develop


For development purposes, it is also useful to know `how to run the system locally`_.

.. _how to run the system locally: INSTALL.rst

Lifecycle
=========

The life of a bot starts by executing ``plebnet setup`` command, which prepares the initial configuration, starts an IRC bot, and creates a cronjob running ``plebnet check`` command every 5 minutes.

The whole lifecycle is then managed by the ``check`` command. First, it ensures Tribler is running. Then it selects a candidate VPS provider and a specific server configuration for the next generation, and calculates the price. One of the pre-defined market strategies is used to convert obtained MB tokens to Bitcoin. Once enough resources are earned, it purchases the selected VPS and VPN options using Cloudomate.

Finally, it connects to the purchased server over SSH, downloads the latest source code from GitHub, install required dependencies, sets up VPN, and runs ``plebnet setup`` to bring the child to life. At that moment, the parent selects a new candidate VPS and continues to maximize its offspring until the end of its own contract expiration.


Reinforcement Learning
======================
The choice of the next VPS to buy is dictated by the Q-Learning technique.

.. TODO: what can we learn about providers? VPS option can be out of stock/Cloudomate broken/provider IP subnet blocked/find most efficient configurations

**What is Q-Learning?**

Q-Learning is a reinforcement learning technique. The aim of this technique
is to learn how to act in the environment. The decision process is based on a data structure called Q-Table, which encodes rewards given by the environment when specific actions are performed in different states.

The values in Q-Table are updated as follows:

.. image:: http://latex.codecogs.com/gif.latex?Q_%7Bnew%7D%28s_%7Bt%7D%2Ca_%7Bt%7D%29%5Cleftarrow%20%281-lr%29&plus;lr*%28reward%20&plus;discount%20*%5Cmax_%7Ba%7D%28s_%7Bt&plus;1%7D%2Ca%29%29

``discount`` is a discount factor (how important gains of future steps are)

``lr`` is a learning rate

``st`` is a current state

``s(t+1)`` is a next step

**Reinforcement Mappings**

We define a few mappings which are used in a reinforcement learning jargon:

- ``states`` - VPS offers

- ``environment`` – transition matrix between states. This determines what reinforcement we will get by choosing a certain transition. Initially all 0s.

- ``current_state`` – current VPS option

**Initial values**

Initial values for Q-Table are calculated according to the formula below:

.. image:: http://latex.codecogs.com/gif.latex?%5Cfrac%7B1%7D%7Bprice%5E3%7D*%20bandwidth


**How does it work in Dollynator?**

In Dollynator we use our own variation of Q-Learning. As we are not fully aware of the environment and our reinforcements for each state, we try to learn them on the go.

Environment is getting updated by each try of replication:

- when a node manages to buy a new option and replicate, environment is updated positively (all transitions leading to ``current_state``)

- when nodes fails to buy an option, environment is updated negatively (the transition between ``current_state`` and the chosen failed state)

After updating the environment values, Q-Table is recalculated one more time to find the action maximizing our possible gains for each state.

**What is passed to the child?**

- its state (provider name + option name)

- name (a unique id)

- tree of replications (a path to the root node)

- providers_offers (all VPS offers for all providers)

- current Q-Table

**Final remarks about reinforcement learning**

To choose an option from Q-Table we use an exponential distribution with lambda converging decreasingly to 1. As lambda is changing with number of replications, this process is similar to **simulated annealing**.

The current version is using a simple formula to choose which kth best option to choose:

.. TODO: state that this is a formula for lambda

.. image:: http://latex.codecogs.com/gif.latex?%5Cleft%20%5Clfloor%201%20-%20%5Cfrac%7B1%7D%7Bno%5C_replications%20&plus;%203%7D%20%5Cright%20%5Crfloor

Market Strategies
=================

The bot has different options for market strategies that can be configured in the configuration file located at ``~/.config/plebnet_setup.cfg``. The used strategy can be specified under the ``strategies`` section in the ``name`` parameter. Possible options are ``last_day_sell``, ``constant_sell``, and ``simple_moving_average``. If it is not configured, ``last_day_sell`` will by applied by default.

There are two main types of strategies to sell the gained reputation for Bitcoin: 

- Blind Strategies focus only on replication independently of the current value of reputation.
- Orderbook-based Strategies focus on getting the most value of the gained reputation, using the history of transactions and having endless options of possible algorithms to use to decide when to sell and when to hold on to the reputation.

Blind Strategies
----------------

Dollynator currently has two options for Blind Strategies: LastDaySell and ConstantSell. Both of the strategies try to obtain enough Bitcoin to lease a certain amount of VPS to replicate to. This number can be configured in the ``vps_count`` parameter in the ``strategy`` section of the configuration file. If it is not configured, ``1`` will be used by default.

LastDaySell waits until there is one day left until the expiration of the current VPS lease and then places an order on the market selling all available reputation for the amount of Bitcoin needed for the configured number of replications. This order is updated hourly with the new income.

ConstantSell, as soon as it is first called, places an order on the market selling all available reputation for the amount of Bitcoin needed for the configured number of replications. This order is updated hourly with the new income.

Orderbook-based Strategies
-------------------------

Dollynator has one Orderbook-based Strategy: SimpleMovingAverage. This strategy tries to get the most of the market by evaluating the current price (the price of the last transaction) against a simple moving average of 30 periods, using days as periods.

This strategy accumulates reputation while the market is not favorable to selling - when the current price is lower than the moving average. It will accumulate up until a maximum of 3 days worth of reputation. When this maximum is reached, even if the market is not favorable, reputation is sold at production rate - the bot waits until the end of the 4th day of accumulation and then places an order selling a full day's worth of reputation.

If the market is favorable - the current price is higher than the moving average - it will evaluate how much higher it is. To do this, the strategy uses the standard deviation of the moving average.

- If it is not above the moving average plus twice the standard deviation, only a full day's worth of reputation is sold.

- If it is between this value and the moving average plus three times the standard deviation, it will sell two days' worth of reputation.

- If it is higher than the moving average plus three times the standard deviation, it will sell three days' worth of reputation.

This strategy doesn't assume market liquidity - even though all placed orders are market orders (orders placed at the last price), it checks if the last token sell was fulfilled completely, only partially, or not at all, and takes that into account for the next iteration. 

If the bot could not gather any history of market transactions, this strategy will replace itself with LastDaySell. 

Continuous Procurement Bot
==========================

In case of insufficient market liquidity, it might be needed to artificially boost MB demand by selling Bitcoin on the market. This is where **buybot** comes into play. It periodically lists all bids on the market, orders them by price and places asks matching the amount and price of bids exactly. It is also possible to make a limit order, so only asks for the bids of price less or equal the limit price would be placed.

.. code-block:: console

    Usage: ./buybot.py <limit price>


Visualization
==============

While the network is fully autonomous, there is a desire to observe its evolution over time. It is possible to communicate with the living bots over an IRC channel defined in ``plebnet_setup.cfg``, using a few simple commands implemented in ``ircbot.py``. Note that all commands only serve for retriving information (e.g. amount of data uploaded, wallet balance, etc.) and do not allow to change the bot's state.

**Plebnet Vision** is a tool allowing to track the state of the botnet over time and visualize the family tree of the whole network. The ``tracker`` module periodically requests the state of all bots and stores it into a file. The ``vision`` module is then a Flask web server which constructs a network graph and generates charts showing how the amount of uploaded and downloaded data, number of Tribler market matchmakers, and MB balance changed over time.


.. image:: https://user-images.githubusercontent.com/1707075/48701343-8d4a4a00-ebee-11e8-87d6-0aecb94caf76.gif
    :width: 60%

After installing the required dependencies, the Flask server and the tracker bot can be started by:

::

    python tools/vision/app_py.py

The HTTP server is running on the port ``5500``.

.. |jenkins_build| image:: https://jenkins-ci.tribler.org/job/GH_PlebNet/badge/icon
    :target: https://jenkins-ci.tribler.org/job/GH_PlebNet
    :alt: Build status on Jenkins

.. _Cloudomate: https://github.com/Tribler/cloudomate
.. _Tribler: https://github.com/Tribler/tribler

Future Work
===========

- Gossip learning protocol using IPv8 overlay: enable collective learning by sharing QTable updates with a secure message authentication
- Q-Table for VPN selection: learn which VPN works the best and which VPS providers ignore DMCA notices and thus do not require VPN
- Market strategies based on other financial analysis' (i.e: other moving averages may be interesting)
- Market strategy based on deep learning
- Explore additional sources of income: Bitcoin donations, torrent seeding...

