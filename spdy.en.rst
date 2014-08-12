SPDY Implementation
+++++++++++++++++++

Configuration
=============

SPDY is configured in the HTTP proxy port definition. It uses a generic mechanism to specify which transport level protocols are accepted on the port. The key ``proto`` is used with a semi-colon separated list of accepted protocols. The basic values are

*  ``http/0.9``
*  ``http/1.0``
*  ``http/1.1``
*  ``http/2``
*  ``spdy/3``
*  ``spdy/3.1``

For convenience some psuedo values are defined. Each of them is identical in effect to explicitly listing its component base protocols.

*  ``http`` means ``http/0.9;http/1.0;http/1.1``
*  ``http2`` means ``http/2``
*  ``spdy`` means ``spdy/3;spdy/3.1``
*  ``all`` means ``http;http2;spdy``

Transport level security (TLS) is defined separately as is already done by adding the ``ssl`` key to the proxy port definition. If ``ssl`` is specified then TLS is required.

The default value for ``proto`` is ``all``.

.. topic:: Example

   Enable all protocols for port 8043, non-encrypted. ::

      8043

.. topic:: Example

   Enable just SPDY 3.1 and HTTP 2, encrypted for port 8043. ::

      8043:proto=spdy/3.1;http/2:ssl

.. topic:: Example

   Enable only HTTP traffic on port 8043. ::

      proto=http;8043

.. note:: The common term "https" really means "http with TLS" and is treated that way for this configuration.

Implementation
==============



Basic Mechanisms
----------------

Accepting incoming connections is done at two layers. The lower layer, the *net accept* layer, handles the TCP level connection and operating system interaction. This is the layer that calls ``accept()`` on the socket. Above the net layer is the *session accept* layer which intermediates between the net layer and the standard ATS data structures.

For each thread that does accepts on a listening socket, an net acceptor is created (an instance or subclass of :cpp:class:`NetAccept`). If the thread is not a dedicated accept thread the net acceptor is treated just as another continuation attached to the file descriptor with the continuation handler set to ``acceptEvent``. When the file descriptor (the listening socket) become ready this handler is called and the connection accepted. In this case the call to ``accept()`` is non-blocking.

For dedicated accept threads the handler is simply a loop that blocks on the call to ``accept()`` and when that returns the connection is accepted.

Attached to each net acceptor as its action is a continuation that is a session acceptor. This will be signaled for each successful TCP connection accept.

The primary work of the net acceptor in accepting a connection is to create a network virtual connection ("NetVC") attached to the newly created socket. The basic actions for this are

#. Create the NetVC.
#. Set the NetVC handler to an accept handler (e.g. ``UnixNetVConnection::acceptEvent``).
#. Set the NetVC action to the net acceptor action.
#. Schedule ``EVENT_IMMEDIATE`` for the NetVC.

When the NetVC recieves the event, it

#. Does internal housekeeping (e.g. register its I/O with the NetHandler)
#. Set its continuation handler to its main event handler.
#. Send ``NET_EVENT_ACCEPT`` to its action.

The NetVC action should be a session acceptor which it got from the net acceptor and the receipt of the ``NET_EVENT_ACCEPT`` is the entry point at which the session acceptor becomes involved.

Session Transport Probing
-------------------------

In the general case we have a first level session acceptor and a set of second level session acceptors. The purpose of the first level is to

#. Data sniff to determine the transport for the session.
#. Call the appropriate second level session acceptor to do the actual session accept.

In the case of a TLS connection we have the additional complication that `NPN <https://technotes.googlecode.com/git/nextprotoneg.html>_` negotiations can also occur. In this case the TLS handling is done first and if NPN is successful, no additional session protocol probing is done. If not, then the TLS handshake is done and the connection is subsequently handled as in the non-TLS case. This means that currently plugins can register additional NPN session protocols for TLS connections but not for non-TLS connections.

The protocol probe requires reading data from the socket to sniff, but that data can't be discarded because the probe is just sniffing, not processing. To deal with this the session acceptors have been changed to take an ``MIOBuffer`` which contains the sniffed data so that it can be processed by the base session acceptor. An additional complication is that the probe shuts down I/O on the client socket after sniffing so the session acceptor (and likely the HTTP state machine) will need to restart the I/O after the sniffed data runs out.

Process Start Initialization
----------------------------

During process start the session acceptors are created first in ``init_HttpProxyServer``. This is done early in the process start sequence so that plugins can get access to the session acceptors before any connections are accepted. The lifecycle hook ``TS_LIFECYCLE_EVENT_PORTS_INITIALIZED`` is called immediately after the session acceptors are created.

At a later point in the process start the net acceptors are created which immediately start to accept incoming connections. After all of the net acceptors are online the hook ``TS_LIFECYCLE_EVENT_PORTS_READY`` is invoked.

The set of protocol for a connection are represented as a bit vector. The bit indices are defined in by :c:type:`TSProtoType`.

SPDY
----

SPDY is effectively a multiplexer and it is handled that way inside Traffic Server. The SPDY state machine controls the client socket and performs the I/O on it. For each transaction the SPDY state machine creates a virtual connection to which it attaches an ``HttpClientSession``. When a request is transmitted over the virtual connection the ``HttpClientSession`` creates the HTTP state machine to process the transaction in the same way as a normal HTTP transaction.

.. figure:: images/ats-spdy-session-data-flow.png
   :align: center

   SPDY data flow

Building with SPDY
------------------

If SPDY isn't available via a package you must download the source from `here <http://tatsuhiro-t.github.io/spdylay/>`_. There is documentation at the link for the build procedure.
If you build it you will probably have to change your package config path to include the package information for the SPDY library.

Ars Technica
============

.. c:type:: TSProtoType

   An enumeration that defines the bit indices for network protocols.
   
.. cpp:class:: ProtocolProbeSessionAccept

   A wrapper acceptor object that contains the actual protocol specific acceptors.

.. cpp:class:: NetAccept

   The base class for net acceptors.
   
