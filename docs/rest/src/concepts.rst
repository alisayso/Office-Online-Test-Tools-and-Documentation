
..  _key concepts:
..  _Concepts:

Key concepts
============

The following concepts are referred to extensively within this documentation, and understanding them is important to
understanding the requirements for integration with WOPI clients such as |wac| and |Office iOS|.


..  glossary::

    File ID
        A File ID is a string that represents a file or folder being operated on via WOPI operations.
        A host must issue a unique ID for any file used by a WOPI client. The client will, in turn, include the
        file ID when making requests to the WOPI host. Thus, a host must be able to use the file ID to locate a
        particular file.

        A file ID must:

        * Represent a single file.
        * Be a URL-safe string because IDs are passed in URLs, principally via the :term:`WOPISrc` parameter.
        * Remain the same when the file is edited.
        * Remain the same when the file is moved or renamed.
        * Remain the same when any ancestor container, including the parent container, is renamed.
        * In the case of shared files, the ID for a given file must be the same for every user that accesses the
          file.

        Note that the file ID is not provided to a WOPI client directly. Rather, it is passed as part of the
        :term:`WOPISrc` value.

        ..  admonition:: |wac| Tip

            See :ref:`Action URLs` and :ref:`Appending WOPISrc` for more information about how the file ID should be
            passed to |wac|.

    Access token
        An access token is a string used by the host to determine the identity and permissions of the issuer of a
        WOPI request.

        The WOPI host that stores the file has the information about user permissions, not the WOPI client. For this
        reason, the WOPI host must provide an access token that the client will then pass back to it on subsequent
        WOPI requests. When the WOPI host receives the token, it either validates it, or responds with an
        appropriate HTTP status code if the token is invalid or unauthorized.

        ..  admonition:: |wac| Tip

            In |wac|, an access token is generated by the host and passed to the client using the
            ``access_token`` parameter before the first WOPI request. See :ref:`Action URLs` and :ref:`Passing access
            tokens securely` for more information about how access tokens should be passed to |wac|.

        A WOPI client requires no understanding of the format or content of the token; the WOPI client simply includes
        it in WOPI requests and expects the host to validate it. However, WOPI access tokens must adhere to the
        following guidelines:

        * Access tokens must be scoped to a single user and resource combination. A WOPI client will never assume that
          an access token issued for a particular user/resource combination is valid for a different user/resource
          combination.
        * Access tokens must be valid for the :ref:`user permissions <permissions>` that are provided by the host in
          the :ref:`CheckFileInfo` response. For example, if the :wopi:action:`view` action is invoked, and the
          :term:`UserCanWrite` property is set to ``true`` in the :ref:`CheckFileInfo` response, then the client may
          re-use that token when transitioning to edit mode. Thus, a WOPI client will expect that any access token
          is valid for operations that the user has permissions to perform. If a host wishes to issue access tokens
          that are more narrowly scoped, then the :ref:`user permissions properties <permissions>` in the
          :ref:`CheckFileInfo` response must reflect the permissions that the token provides.
        * Access tokens should expire (become invalid) automatically after a period of time. Hosts can use the
          :term:`access_token_ttl` property to specify when an access token expires.

        ..  important::
            WOPI clients expect that an access token will remain valid until it expires (as indicated by the
            :term:`access_token_ttl` value). Hosts should not revoke access tokens as a standard part of their
            operations; tokens should only be revoked if a user's permissions have changed or been revoked.

            If hosts revoke access tokens regularly, users may experience strange behavior depending on the WOPI
            client. For example, in some cases |wac| will time out a user's session and present them with a
            dialog asking if they'd like to refresh their session. If the access token is not yet expired based on the
            :term:`access_token_ttl`, |wac| will refresh the session using the existing access token,
            assuming that it is still valid.

            There are a number of cases like this, and WOPI clients regularly make such assumptions. For this reason,
            access tokens should remain valid until their expiry.

        ..  include:: /_fragments/access_token_handling_warning.rst

    access_token_ttl
        The access_token_ttl property tells a WOPI client when an access token expires, represented as the
        number of milliseconds since January 1, 1970 (the date epoch in JavaScript). Despite its misleading name,
        it does *not* represent a duration of time for which the access token is valid. The access_token_ttl is never
        used by itself; it is always attached to a specific access token.

        ..  admonition:: |wac| Tip

            To prevent data loss for users, |wac| will prompt users to save and refresh their sessions if the
            access token for their session is close to expiring. In order to do this, |wac| needs to know when
            the access token will expire, which it determines based on the access_token_ttl value.

        Hosts can set the access_token_ttl value to ``0``. This will effectively tell the client that the
        token expiry is either infinite or unknown. In this case, clients may disable any UI prompting users
        to refresh their sessions. This can lead to unexpected data loss due to access token expiry, so specifying a
        value for access_token_ttl is strongly recommended.

        ..  note::

            Future updates to the WOPI protocol may rename this parameter so its name is less confusing.


    Lock
        A lock is a conceptual construct that is associated with a file. Locks serve two purposes in WOPI:

        1.  First, a lock prevents anyone that does not have a valid lock ID from making changes to a file. A WOPI
            client will lock files prior to editing them to prevent other entities from changing the file while the
            client is also editing them.
        2.  A lock is also used to store a small bit of temporary data associated with a file. This metadata is called
            the *lock ID* and is a string with a maximum length of 1024 ASCII characters (see
            :ref:`note on lock ID lengths<lock length>`). WOPI clients can use this metadata for a variety of
            purposes, but hosts do not need any knowledge or understanding of the contents of the lock ID. Hosts must
            treat it as an opaque string.

        Therefore, WOPI locks must:

        * Be associated with a single file.
        * Contain a lock ID of maximum length 1024 ASCII characters.
        * Prevent all changes to that file unless a proper lock ID is provided.
        * Expire after 30 minutes unless refreshed (see :ref:`RefreshLock`).
        * *Not* be associated with a particular user.

        All WOPI operations that modify files, such as :ref:`PutFile`, will include a lock ID as a parameter in their
        request. Usually the ID will be passed in the **X-WOPI-Lock** request header (but not always;
        :ref:`UnlockAndRelock` is an exception). WOPI requires that hosts compare the lock ID passed in a WOPI
        request with the lock ID currently on a file and respond appropriately when the lock IDs do not match. In
        particular, WOPI clients expect that when a lock ID does *not* match the current lock, the host will send
        back the current lock ID in the **X-WOPI-Lock** response header. This behavior is critical, because WOPI
        clients will use the current lock ID in order to determine what further WOPI calls to make to the host.

        It is important to note that WOPI locks are *not* user-owned. In other words, a WOPI client may execute
        lock-related operations using multiple access tokens, and hosts are expected to execute those operations as
        long as they are valid as described in this documentation. For example, a WOPI host may receive a :ref:`Lock`
        call with an access token that belongs to User A. The host may later receive an :ref:`Unlock` call
        with an access token that belongs to User B. As long as User B has rights to edit the file, and the
        **X-WOPI-Lock** request header matches the lock ID, the :ref:`Unlock` request should be honored.

        ..  admonition:: |wac| Tip

            WOPI defines a :ref:`GetLock` operation. However, it is not currently called by |wac|. Instead,
            |wac| will often execute lock-related operations on files with missing or known incorrect lock IDs
            and expects the host to provide the current lock ID in its WOPI response. Typically the :ref:`Unlock` and
            :ref:`RefreshLock` operations are used for this purpose, but other lock-related operations may be used.

            In the future, |wac| will call :ref:`GetLock` for this purpose if the host sets the
            :term:`SupportsGetLock` property in :ref:`CheckFileInfo`.

        The specific conditions for each response are covered in the documentation for each of the
        following lock-related WOPI operations:

        * :ref:`Lock`
        * :ref:`RefreshLock`
        * :ref:`Unlock`
        * :ref:`UnlockAndRelock`
        * :ref:`PutFile`

        ..  _lock length:

        ..  note::
            Lock ID lengths are currently less than 256 ASCII characters. However, we anticipate requiring longer
            lock IDs to support future WOPI integration scenarios, so we have increased the limit to 1024
            ASCII characters. Hosts must indicate that they support lock IDs of this length using the
            :term:`SupportsExtendedLockLength` property in :ref:`CheckFileInfo`.


    Share URL
        A Share URL is a URL to a webpage that is suitable for viewing a shared WOPI file or container. The URL should be
        appropriate for being launched in a web browser, but the experience is defined by the host. For example, 
        the host may choose to have the URL navigate to the host's browse experience or to a preview of the file 
        using Office Online or another file previewer. 

        A host may support different types of Share URLs that may be used for different purposes. For example, a
        particular Share URL type may not allow users to edit the file by using the Share URL. The list of possible
        types are defined under the :term:`SupportedShareUrlTypes` property.

    WOPISrc
        The WOPISrc (*WOPI Source*) is the URL used to execute WOPI operations on a file. It is a combination of the
        :ref:`Files endpoint` URL for the host along with a particular :term:`file ID`. The WOPISrc does *not*
        include an :term:`access token`.

        The WOPISrc is needed beyond just a file ID so that a WOPI client can know what URL to call back to when
        executing WOPI operations on a file. In practice, the WOPISrc and a :term:`file ID` are synonymous, since
        WOPI client typically work with the WOPISrc itself, not the raw :term:`file ID`.

        ..  admonition:: |wac| Tip

            See :ref:`Appending WOPISrc` for more details on how the WOPISrc is constructed and passed to
            |wac|.

    Container
        |stub-icon| Not yet documented.

    Root Container
        |stub-icon| Not yet documented.

    Ecosystem
        |stub-icon| Not yet documented.

    Bootstrapper
        |stub-icon| Not yet documented.
