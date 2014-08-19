SSL API
=======

Config File Addition
--------------------

.. c:type:: ssl_multicert.config 

This file contains references to certificate files, addresses and keys.  It is loaded into a table keyed by IP address and host name (possibly wildcarded) at the start of the Traffic Server process.  The table is used to determine the SSL context to use when proxying SSL connections.  

This feature set adds the ``action`` attribute to :c:type:`ssl_multicert.config`.  If ``action`` is set to ``tunnel1``, the connection that matches that line will blind tunnel the SSL connection rather than proxying it.  The `default ssl_multicert.config <https://github.com/shinrich/trafficserver/blob/ts-3006/proxy/config/ssl_multicert.config.default>`_ has been updated to show an example of the ``action`` attribute.

SSL Hooks
---------

In all cases, the hook callback has the following signature.

.. c:function:: int SSL_callback(TSCont contp, TSEvent event, void *edata)

The :arg:`edata` parameter is a TSSslVConn object.

The following actions are valid from these callbacks.

* Fetch the SSL object associated with the connection - :c:func:`TSSslVConnObjectGet`
* Set a connection operation - :c:func:`TSSslVConnOpSet`
* Reenable the ssl connection - :c:func:`TSSslVConnReenable`
* Find SSL context by name - :c:func:`TSSslCertFindByName`
* Find SSL context by address - :c:func:`TSSslCertFindByAddress`


.. c:type:: TS_SSL_CLIENT_PRE_HANDSHAKE_HOOK

This hook is invoked after the client has connected to ATS and before the SSL handshake is started, i.e., before any bytes have been read from the client. The data for the callback is a TSSslVConn instance which represents the client connection. There is no HTTP transaction as no headers have been read.

.. c:type:: TS_SSL_SNI_HOOK

This hook is called if the client provides SNI information in the SSL handshake. If called it will always be called after ``TS_SSL_CLIENT_PRE_HANDSHAKE_HOOK``.

The following action is valid from this hook.

* Get the client specified server name - :c:func:`TSSslVConnServernameGet`

The Traffic Server core first executes logic in the SNI callback to store the servername in the TSSslVConn object and  to evaluate the settings in the 
:c:type:`ssl_multicert.config` file based on the server name.  Then the core SNI
callback executes the plugin registered SNI callback code.  The plugin callback can access the servername by calling :c:func:`TSSslVConnServernameGet`.

If server running Traffic server has the appropriate openSSL patch installed, the SNI callback can return ``SSL_TLSEXT_ERR_READ_AGAIN`` to stop the SSL handshake processing.  This results in ``SSL_accept`` returning ``SSL_ERROR_WANT_SNI_RESOLVE`` before completing the SSL handshake (only the client hello message will have been received).  Additional processing could reenable the virtual connection causing the ``SSL_accept`` to be called again to complete the handshake exchange.  In the case of a blind tunnel conversion, the SSL handshake will never be completed by Traffic Server.

The plugin callbacks can halt the SSL handshake processing by not reenabling the connection (i.e., by not calling :c:func:`TSSslVConnReenable`).  If a plugin SNI callback does not reenable the connection, the global callback will return ``SSL_TLSEXT_ERR_READ_AGAIN``.

Without the openSSL patch, the handshake processing in ``SSL_accept`` will not
be stopped even if the SNI callback does not reenable the connection.

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

.. c:type:: TSSslVConnObject

   The SSL (per connection) object.  This is an opaque type that can be cast to the appropriate type (SSL * for the openSSL library).  

.. c:type:: TSSslContext

   Corresponds to the SSL_CTX * value in openssl.

.. c:type:: TSCont

   Stub

.. c:type:: TSEvent

   Stub

.. c:type:: TSReturnCode

   Stub

Utility Functions
-----------------

.. c:function:: TSReturnCode TSSslVConnOpSet(TSSslVConn svc, TSSslVConnOp op)

   Set the SSL connection :arg:`svc` to have the operation :arg:`op` performed on it.

.. c:function:: void TSSslVConnReenable(TSSslVConn svc)

   Reenable the SSL connection :arg:`svc`. If a plugin hook is called, ATS processing on that connnection will not resume until this is invoked for that connection.

.. c:function:: TSSslVConnObject TSSslVConnObjectGet(TSSslVConn svc)

   Get the SSL (per connection) object from the SSl connection :arg:`svc`.

.. c:function:: char * TSSslVConnServernameGet(TSSslVConn svc)

   Get the name of the server as specified by the client via the SNI extensions.  If no server name is specified, NULL is returned.

.. c:function:: TSSslContext TSSslCertFindByName(TSSslVConn svc, char *name)

   Look for a SSL context create from the :c:type`ssl_multicert.config` file.  Use the server name to search.

.. c:function:: TSSslContext TSSslCertFindByAddress(TSSslVConn svc, struct sockaddr const*)

   Look for a SSL context create from the :c:type:`ssl_multicert.config` file.  Use the server address to search.

Example Uses
------------

Three examples have been added to the code base illustrating how these additions can be used.

Example one is `ssl-preaccept <https://github.com/shinrich/trafficserver/blob/ts-3006/example/ssl-preaccept/ssl-preaccept.cc>`_ which uses the new :c:type:`TS_SSL_CLIENT_PRE_HANDSHAKE_HOOK` to implement a blind tunnel if the client IP address matches one of the ranges in the config file.  Function ``CB_Pre_Accept`` contains the interesting bits.

Example two, `ssl-sni-whitelist <https://github.com/shinrich/trafficserver/blob/ts-3006/example/ssl-sni-whitelist/ssl-sni-whitelist.cc>`_,  uses the SNI callback.  It takes the servername and destination address to lookup SSL context information loaded from the :c:type:`ssl_multicert.config` file.  If no SSL context can be found, the callback sets the connection to use blind tunnelling.  The information in the :c:type:`ssl_multicert.config` file whitelists the SSL sites to be proxied. Function ``CB_servername_whitelist`` is the callback function.

Example three is `ssl-sni <https://github.com/shinrich/trafficserver/blob/ts-3006/example/ssl-sni/ssl-sni.cc>`_. This example is not a useful real world scenario but a test that exercises the new functions added in this feature addition.  This example installs a SNI callback (``CB_servername``).  The callback tests if the servername ends in ``facebook.com``.  If it does, the callback sets up a blind tunnel.  Otherwise, if the servername is ``www.yahoo.com``, the callback looks up the SSL context loaded for ``safelyfiled.com`` and sets that context for the connection.
