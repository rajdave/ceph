=========
MonClient
=========

Concepts
--------

*Messenger*
  See src/msg/Messenger.h

  Handles sending and receipt of messages on behalf of the MonClient.  The MonClient uses
  a messenger called "messenger", which handles all traffic to monitors.

*Dispatcher*
  See src/msg/Dispatcher.h

  MonClient implements the Dispatcher interface.  Of particular note is ms_dispatch,
  which serves as the entry point for messages received via the messenger. The monc_lock
  is held during ms_dispatch if the message type is valid.

*MonMap*
  See src/mon/MonMap.h

  The MonClient can track monitors it is connected to using information
  stored in the monmap, which holds inst, address, and name data for monitors.
  Maps are numbered by *epoch* (epoch_t).

*MonClientState*
  See src/mon/MonClient.h

  Defined in the MonClient.h header file, this is the status of a connection
  to a monitor that a MonClient is communicating with.

*MonConnection*
  See src/mon/MonClient.h

  Defined in the MonClient.h header file, this holds information about connections
  this ``MonClient`` is managing, including ``ConnectionRef`` (con), ``State`` (state - status
  of connection), and ``AuthClientHandler`` pointer (auth - the authentication handler
  for this connection).

*Message Types*
  See src/messages/*

  There are various types of messages accepted and sent by a MonClient which are used
  for communication with a monitor.

*Hunting*
  The ``MonClient`` periodically has to hunt for a new monitor to connect to if either
  the monitor it is currently communicating with goes down or if it is told to reset
  or reconnect.

  Hunting primarily takes place through a call to ``_reopen_session()`` within the ``MonClient``
  functions. In this function, either a specific monitor is given or the ``MonClient`` creates
  connections with multiple monitors at the same time. After this, all the old waiting
  for session and other messages are discarded. The MonClient sends MAuth(s) (a type of
  message) to the monitor(s) in an attempt to authenticate.
  The lock for the MonClient (monc_lock) is held while reopen session is working, but will be
  relinquished when the caller returns.

  Asynchronously, the monitor(s) which received the ``MAuth`` will process that message and should
  send an ``MAuthReply``. When the MonClient receives this message in ``ms_dispatch``, it forwards it
  to the handler (``handle_auth``). The ``handle_auth`` function will process the message, and, if
  successful, sets the ``active_con``, signals the ones waiting for the authand finishes the hunting process.

Overview
--------
  See src/mon/MonClient.cc

  The MonClient represents the communication interface by which processes may
  send and receive information from monitors, and generally handles the connection,
  monmap retrieval, and monitor hunting processes.


Key Elements
------------

*active_con*
  This is the ``MonConnection`` pointer that points to data about the in session connection. It will
  only be valid if there is currently a valid session with a monitor. It holds the current connection
  reference (con), the ``AuthClientHandler`` pointer, and the current state of the in session connection
  (state).

*pending_cons*
  This is a map from ``entity_addr_t`` addresses (for monitors) to pending MonConnections (not in session).
  This map will be populated by ``_reopen_session()`` with various MonConnections holding data about the connections.
  This information is used during the authentication process, then is discarded once a session has been
  established.

*monc_lock*
  This is the mutex for the MonClient.

*auth_cond*
  This condition variable is used to wait for a successful authentication with a monitor
  (notably by the function "authenticate"), which is signalled by handle_auth.

*auth_map*
  This is a set of ``entity_addr_t``, ``AuthClientHandler`` pointer pairs that is used during the hunting
  process to coordinate authentication with multiple monitors at a time. Once the hunting process
  is completed and a session has been established, the variable auth is set and is used for other
  authentication processes during a session. This mapping is used only by the session establishing
  processes of handle_auth.

*want_monmap*
  This is set by the constructor and used to wait for a monmap. It is set false by the handler
  for a monmap message (``handle_monmap()``).

*waiting_for_session*
  When the ``MonClient`` is not yet authenticated to a monitor (``active_con`` is not valid yet),
  all messages sent will be pushed onto this list. Whenever a session is established in ``handle_auth()``,
  the MonClient then sends all messages on this list to that monitor.
  This list is cleared by ``_reopen_session()``.

*subs*
  This group of maps (sub_sent, sub_new) and functions (_sub_want, _sub_got, _sub_unwant,
  _renew_subs), are used to manage the "subscriptions" of this MonClient. That is, subscriptions
  are registered and removed from the mappings by the functions ``_sub_want()``, ``_sub_got()``, and
  ``_sub_unwant()``. The function _renew_subs sends out a ``MMonSubscribe`` message to the current monitor
  with the sub_new subscriptions.
  These are sent out periodically throughout the code, but most noteably on a timely basis
  by the function tick().
