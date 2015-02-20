SSL Context Initialize Plugin 
=============================

OpenSsl has a vast number of callbacks and functions that can be associated with SSL contexts (SSL_CTX * ). ATS uses some of those callbacks (e.g. for SNI, certificate selection, and session cache), but it does not use all of them.  The ATS user may wish to take advantage of some of the additional openssl features that are available associated with the SSL_CTX object.  For example, you can use SSL_CTX_set_generate_session_id to set a callback to provide some constraints on the session ID name selection.  Using this callback would not conflict with the session callbacks that ATS uses for session cache support.

To effectively add callbacks to the SSL_CTX object, the plugin writer will need access to the SSL_CTX object right after it is created and before it is used.  The plugin writer can use the TS_LIFECYCLE_SERVER_SSL_CTX_INITIALIZED_HOOK and TS_LIFECYCLE_CLIENT_SSL_CTX_INITIALIZED_HOOK hooks to initialize the server SSL_CTX and client SSL_CTX contexts respectively.


The lifecycle hook callback has the following signature.

.. c:function:: int SSL_context_callback(TSCont contp, TSEvent event, void *edata)

The :arg:`edata` parameter is a TSSslContext object.



