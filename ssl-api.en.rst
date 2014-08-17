SSL API
=======

TS_SSL_CLIENT_PRE_HANDSHAKE_HOOK
--------------------------------

This is a global hook.

This hook is invoked after the client has connected to ATS and before the SSL handshake is started, i.e., before any bytes have been read from the client. The data for the callback is a :c:type:`TSSslVConn` instance which represents the client connection. There is no HTTP transaction as no headers have been read.

The actions that are valid from this callback are

* Get the SSL context - :c:func:`TSSslVConnContextGet`
* Set the SSL context - :c:func:`TSSslVConnContextSet`
* Set a connection operation - :c:func:`TSSslVConnOpSet`
* Reenable the ssl connection - :c:func:`TSSslVConnReenable`

TS_SSL_CLIENT_SNI_HOOK
----------------------

This hook requires a patch to the openSSL library to make it asynchronous. This hook is called if the client provides SNI information in the SSL handshake. If called it will always be called after ``TS_SSL_CLIENT_PRE_HANDSHAKE_HOOK``. Any ATS internal SNI configuration actions will be performed before this hook is invoked which allows the plugin to override that configuration.

Types
-----

.. c:type:: TSVConn

   A virtual connection.

.. c:type:: TSSslVConn

   An SSL connection. It is a subclass of :c:type:`TSVConn` and can be used as one.

.. c:type:: TSSslVConnOp

   An enumeration specifying the various operations that can be done for an SSL connection.

   ``TS_SSL_HOOK_OP_TERMINATE``
      The SSL connection will be terminated as soon as possible. This will normally mean simply closing the TCP connection.
   ``TS_SSL_HOOK_OP_TUNNEL``
      No further SSL or HTTP processing will be done, the connection will be blind tunneled to its destination.

.. c:type:: TSSslObject

   The SSL (per connection) object. This can be cast directly to the openSSL type ``SSL``.

Utility Functions
-----------------

.. c:function:: void* TSSslVConnContextGet(TSSslVConn svc)

   Get the SSL context for the connection :arg:`svc`.

   .. note:: that if ATS is configured to use the SNI information this context may not be used if it is overridden by that configuration.

   .. note:: You can force this context by using :c:func:`TSSslVConnContextSet` and passing the context retrieved by this function.

.. c:function:: bool TSSslVConnContextSet(TSSslVConn svc, void* ssl_ctx)

   Set the SSL context to be used for this conection. This overrides any ATS configuration based actions. In particular configuration actions based on SNI information (e.g., certificate selection). Because this overrides any ATS setup for the context it is the caller's responsibility to set any required or desired values in :arg:`ssl_ctx`.

   .. note:
      An SSL context is effectively a global object. The same one may be used for multiple connections and therefore modifying one can have effects on other transactions.

.. c:function:: bool TSSslVConnOpSet(TSSslVConn svc, TSSslVConnOp op)

   Set the SSL connection :arg:`svc` to have the operation :arg:`op` performed on it.

.. c:function:: void TSSslVConnReenable(TSSslVConn svc)

   Reenable the SSL connection :arg:`svc`. If a plugin hook is called, ATS processing on that connnection will not resume until this is invoked for that connection.

.. c:function:: TSSslObject TSSslObjectGet(TSSslVConn svc)

   Get the SSL (per connection) object from the SSl connection :arg:`svc`.
