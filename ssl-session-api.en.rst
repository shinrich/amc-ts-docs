SSL Session Plugin API
======================

These interfaces enable a plugin to hook into operations on the session cache.  It also enables the plugin to update the session cache based on outside information.

SSL Session Hook
-----------------

The hook is 

.. c:type:: TS_SSL_SESSION_HOOK

This hook is invoked when a change has been made to the ATS session cache or a session has been accessed from the ATS session cache via openssl.


The hook callback has the following signature.

.. c:function:: int SSL_session_callback(TSCont contp, TSEvent event, void *edata)

The :arg:`edata` parameter is a TSSslSessionId object.

The following events be sent to this callback.

.. c:type:: TS_EVENT_SESSION_INSERT 

 This event is sent after a new session has been inserted into the ATS session cache.  The plugin can call :c:func:`TSSslSessionGet` to retrieve the actual session object.  The plugin could communicate information about the new session to other processes or update additional logging and statistics.

.. c:type:: TS_EVENT_SESSION_REMOVE

This event is sent after a session has been removed from the ATS session cache.  The plugin could communicate information about the session removal to other processes or update additional logging and statistics.

.. c:type:: TS_EVENT_SESSION_GET

This event is sent after a session has been fetched from the ATS session cache by a client request.  The plugin could update additional logging and statistics.

The following action is valid from this callback.

* Fetch the SSL session object associated with the SSL session Id - :c:func:`TSSslSessionGet`


Types
-----

.. c:type:: TSSslSessionId

The SSL session ID object.

.. c:type:: TSSslSession

The SSL session object.  It can be cast to the openssl type SSL_SESSION * .

Utility Functions
-----------------

These functions perform the appropriate locking on the session cache to avoid errors.

.. c:function:: TSSslSession TSSslSessionGet(TSSsslSessionId sessionid)

Returns the SSL session object stored in the ATS session cache that corresponds to the sessionid parameter.  If there is no match, it returns NULL.

.. c:function:: TSReturnCode TSSslSessionInsert(TSSslSessionId sessionId, TSSslSession addSession)

This function inserts the session specified by the addSession parameter into the session cache under the sessionId key.

What should happen if there is already an entry for the sessionID?  Overwrite or fail?

.. c:function:: TSReturnCode TSSslSessionRemove(TSSslSessionId sessionId)

This function removes the session entry from the session cache that is keyed by sessionId.

Example Use Case
----------------

A company is deploying a set of ATS boxes as a farm behind a load balancer.  The load balancer does not guarantee that all the requests from a single client are directed to the same ATS box.  Therefore, to maximize session reuse, they want to share session state between boxes.  They have some external communication library to perform this sharing.

They write a plugin that sets the TS_SSL_SESSION_HOOK.  When the hook goes off, the plugin function sends the upate of the session state to the other ATS boxes using its communication library.

The plugin also has a thread that listens for updates and calls TSSSlSessionInsert and TSSslSessionRemove to update the local session cache accordingly.
