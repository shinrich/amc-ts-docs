SSL API
=======

TS_SSL_CLIENT_PRE_HANDSHAKE_HOOK
--------------------------------

This is a global hook.

This hook is invoked after the client has connected to ATS and before the SSL handshake is done. This means before any bytes have been read from the client. The data for the callback is a TSSslVConn instance which represents the client connection. There is no HTTP transaction as no headers have been read.

The actions that are valid from this callback are

* Get the SSL context - :c:func:`TSSslContextGet`
* Set the SSL context - :c:func:`TSSslContextSet`
* Set a connection operation - :c:func:`TSSslVConnOpSet`

TS_SSL_CLIENT_SNI_HOOK
----------------------

This hook requires a patch to the openSSL library to make it asynchronous. This hook is called if the client provides SNI information in the SSL handshake. If called it will always be called after ``TS_SSL_CLIENT_PRE_HANDSHAKE_HOOK``. Any core SNI configuration actions will be performed before this hook is invoked which allows the plugin to override that configuration.

Types
-----

.. c:type:: TSVConn

   A virtual connection.

.. c:type:: TSSslVConn

   An SSL connection. It is a subclass of :c:type:`TSVConn` and can be successfully case to that type.

.. c:type:: TSSslVConnOp

   An enumeration specifying the various operations that can be done for an SSL connection.

   ``TERMINATE``
      The SSL connection will be terminated as soon as possible. This will normally mean simply closing the TCP connection.
   ``TUNNEL``
      No further SSL processing will be done, the connection will be blind tunneled to its destination.

.. c:type:: TSSslObject

   The SSL (per connection) object.

Utility Functions
-----------------

.. c:function:: void* TSSslContextGet(TSSslVConn svc)

   Get the SSL context for the connection :arg:`svc`.

   .. note:: that if ATS is configured to use the SNI information this context may not be used if it is overridden by that configuration.

   .. note:: You can force this context by using :c:func:`TSSslContextSet` and passing the context retrieved by this function.

.. c:function:: bool TSSslContextSet(TSSslVConn svc, void* ssl_ctx)

   Set the SSL context to be used for this conection. This overrides any further configuration, in particular any SNI based configuration. Because this overrides any ATS setup for the context it is the caller's responsibility to set any required or desired values in :arg:`ssl_ctx`.

.. c:function:: bool TSSslVConnOpSet(TSSslVConn svc, TSSslVConnOp op)

   Set the SSL connection :arg:`svc` to have the operation :arg:`op` performed on it.

.. c:function:: void TSSslRenable(TSSslVConn svc)

   Reenable the SSL connection :arg:`svc`. This must be called if a hook is invoked on the SSL connection.

.. c:function:: TSSslObject TSSslObjectGet(TSSslVConn svc)

   Get the SSL (per connection) object from the SSl connection :arg:`svc`.
