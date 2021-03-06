RESTful Bag Store
=================

Overview
--------

This document briefly describes an approach to making `BagIt
<http://en.wikipedia.org/wiki/BagIt>`_ style containers of content
available on the Web so that content can be easily discovered,
retrieved and otherwise annotated. BagIt style directories, or Bags,
are a lightweight mechanism for documenting a set of files and their
fixities in a manifest, and associating them with some generic
metadata.

Work on an a REST API for Bags grew out of conversations started by
the `Educopia Institute <http://www.educopia.org/>`_ in
late 2010. Before diving into the basic design, a few use cases to
describe the problem space will be described.

Use Cases
---------

* `National Digital Newspaper Program <Use%20Cases/NDNP.rst>`_
* `Chronopolis <Use%20Cases/Chronopolis.rst>`_
* `Penn State <Use%20Cases/PennState.rst>`_
* `Archivematica <Use%20Cases/Archivematica.rst>`_

Design
------

This describes the public interface of an endpoint, which could be an entire
service or a project-specific subdirectory on generic content storage system.

Basic Features
--------------

* Pure HTTP
* Authentication and access control are considered a server implementation
  issue using standard HTTP techniques
* Does not require an intelligent server, with a goal of allowing simple
  static servers or cloud services like S3 to be used as compatible, read-only
  implementations

Controversial Points
--------------------

* Implementations MUST support JSON representations of resources, MAY support
  XML and other formats


Structure
~~~~~~~~~

:/changes:
    Feed listing new bags (should the feed list only new bags, or also e.g.
    deleted and modified bags?)

:/bags/:
    Resource listing available bags.

    Responses are returned with pagination:

        :pagination:
            :offset:
                Offset into the list of bags
            :limit:
                Limit on the number of results
            :total_count:
                Total number of bags
            :next:
                Link to the next page of results, if available
            :previous:
                Link to the previous page of results, if available
        :objects:
            List of bags in the following format:
                :href:
                    Location of the bag
                :id:
                    User-assigned bag ID

GETing ``/bags/`` <*BAG_ID*> ``/`` will return a response containing the
following metadata:

    :links:
        List of links to other resources on this server (see below) using the
        following format, as in HTML ``link`` tags (see `RFC 5988
        <http://tools.ietf.org/html/rfc5988>`_ for valid rel values).

        :rel:
            forward link types
        :href:
            URI for linked resource
        :type:
            advisory content type

    :info:
        Parsed values from ``bag-info.txt`` returned as a list of key-value
        pairs

    :bagit:
        Parsed dictionary from ``bagit.txt``

Clients may POST to ``/bags/`` <*BAG_ID*> ``/`` to perform several operations:

    :commit:
        Complete an upload (see "Creating a bag" below)

        Servers *MUST* not include a bag in any public listings until the bag
        has been committed.

    :validate:
        Request that the server validate the bag contents against the manifest

Under ``/bags/`` <*BAG_ID*> ``/`` will be several resources:

    :copies:
        Feed listing alternate locations for this bag by URL

        TODO: specify format

        Mirrors can PUT their location after mirroring this bag. Servers are
        not required to accept these requests.

        TODO: Specify rel types for instances

    :notes:
        Feed containing comments from curators

        TODO: Should this be history?

    :versions:

        GET returns a list of version IDs::

            [
                {
                    "id": "<unique identifier>",
                    "timestamp": "ISO 8601 representation",
                    "name": "<optional symbolic name>"
                },
                …
            ]

        Each version contains two resources:

            :manifest:
                Resource enumerating bag contents. This is a dictionary with two keys:

                :tag:
                    List of tag files as defined in the BagIt specification section
                    1.3 (Terminology)

                :payload:
                    List of payload files as defined in the BagIt specification
                    section 1.3 (Terminology)

                Each list contains dictionaries with the following structure:

                :path:
                    The file's full path relative to the bag root, i.e. ``data/foobar.tiff``

                :checksum:
                    Dictionary of encoded checksum values using the algorithm as the
                    key. This is optional for tag files.

                Example::

                    {
                        "payload": [
                            {
                                "checksum": {
                                    "md5": "00fcbdf37a87dced7b969386efe6e132",
                                    "sha1": "74a272487eb513f2fb3984f2a7028871fcfb069b"
                                },
                                "path": "data/path/to/example.pdf"
                            }
                        ],
                        "tag": [
                            {
                                "path": "bagit.txt"
                            },
                            {
                                "path": "bag-info.txt"
                            },
                            {
                                "path": "manifest-md5.txt"
                            },
                            {
                                "path": "manifest-sha1.txt"
                            }
                        ]
                    }

            :contents:
                Root for access to bag contents: for any file path in the manifest,
                ``/bags/`` <*BAG_ID*> ``/contents/`` <*PATH*> will return the raw
                file.

    :metadata:
        Arbitrary additional metadata files stored in Java-style reversed
        domain prefixed files

        GET returns a simple file list (Atom feed?), allowing clients to
        decide whether they wish to retrieve a file

        The server promised only that the metadata files will be preserved
        with the same level of durability as the bag contents

        Example::

            [
                'gov.loc.exampleProject.backup_history.xml',
                'com.flickr.commons.userComments.json',
                'org.apache.tika.extractedMetadata.xml'
            ]


Versioning
~~~~~~~~~~

The versioning semantics are designed to support the use of version control
systems like Git or Mercurial as storage backends and to allow implementors to
support efficient delta-based replication protocols beyond the scope of this
specification.

* All content within a version *MUST* be immutable but servers *MAY* remove
  old versions as desired. This allows bag copies to be compared simply by
  comparing the source URLs of valid bags.

  This promise of immutability applies only to to the bag contents, including
  the top-level tag files, and includes any file addition or deletion within
  the content directory.
  Metadata files are not versioned to avoid local additions breaking
  replication.

* Arbitrary symbolic names may be provided but *MUST* redirect to the
  appropriate hash value so clients can perform consistent equality checks.


Good HTTP Citizenship
~~~~~~~~~~~~~~~~~~~~~

A summary of relevant points from
`HTTP 1.1 (RFC 2616) <http://www.w3.org/Protocols/rfc2616/rfc2616.html>`_ which
are of particular value for archival and replication:

* Servers *SHOULD* generate Cache-Control headers; clients *MUST* honor them
* Servers *MAY* use HTTP redirects to direct clients to HTTP-accessible
  backend storage for performance reasons
* If available, servers *SHOULD* return ``Content-MD5`` or ``Content-SHA1``
  headers using the hash value from the manifest; clients *SHOULD* validate
  these values if present
* Servers *SHOULD* support entity tags and ``If-None-Match``
* Servers *SHOULD* support HTTP Range to allow clients to resume transfers
* Servers *MAY* provide ``Retry-After`` with HTTP 503 (Service Unavailable)
  to help clients, particularly when the delay is due to content being staged
  from slower archive storage with known latency characteristics
* Clients *MUST* honor HTTP 503 Service Unavailable responses using a provided
  ``Retry-After`` header or using exponential back-off if ``Retry-After`` is not
  provided.
* Servers *SHOULD* support HTTP Pipelining and Keep-Alives to avoid
  performance issues when transferring large numbers of small files
* Servers *SHOULD* return HTTP 410 (Gone) for content which has been removed,
  particularly in the case of old versions for bags which are still present.

Operations
~~~~~~~~~~

For this discussion, it is assumed that servers may return standard HTTP
response code such as 401/403 to indicate that the client needs to
authenticate or lacks permissions to make changes.

Creating a new bag
^^^^^^^^^^^^^^^^^^

    #. Create the container:
        Client POSTs to ``/bags``:
            :id: unique bag identifier
            :version: optional client-suggested version, which the server may
                      choose to use

        Server returns 201 pointing to the upload location, which may be
        the final destination e.g. ``/bags/:id:/versions/:version-id:/`` or
        a temporary location, possibly pointing to a specific back-end storage
        server.

        Clients *MUST* perform all subsequent operations using the
        server-provided location

        Servers *MUST* return 409 Conflict if the ID is already in use

    #. Client PUTs ``bagit.txt`` and ``bag-info.txt`` under ``contents``

    #. Client PUTs one or more manifest files under ``/contents/``

        Servers *MUST* return HTTP 400 if the client has not provided
        ``bagit.txt`` or ``bag-info.txt``

        Clients *MUST* provide the manifest files before committing the bag but
        are free to assemble the bag in any order

    #. Client PUTs data files under ``contents/data/``

        Servers *MUST* return HTTP 400 if the client has not provided at least
        one manifest file or attempts to PUT a file which is not listed in the
        manifest or fails checksum validation

        Clients *MAY* use DELETE or PUT to remove or replace previously uploaded
        files before committing the bag

    #. Client POSTs ``commit`` to the bag location

        Servers *MUST* return HTTP 400 if all of the files which are specified
        in the manifest have not been received


Deleting a bag
^^^^^^^^^^^^^^

    #. Client DELETEs bag location

Replicating a bag
^^^^^^^^^^^^^^^^^

    #. Client GETs ``manifest``
    #. Client GETs each listed content file
    #. Optionally, client performs an AtomPub POST to ``copies`` with the
       public URL of a copy conforming to this specification.

Requesting Server Validation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

    #. Client POSTs operation=validate to ``/bags/`` <*BAG_ID*>
    #. Server returns HTTP 202 Accepted and an initial status resource with
       the following attributes:

       :uri:
           Unique URI which the client can GET to retrieve the current
           status

       :status:
           One of ``In Progress``, ``Failed``, or ``Successful``

       :progress:
           Integer percentage or null if the server does not support
           partial status

       :message:
           Human-readable summary message, which may only be available
           when the operation has completed

       :manifest:
           List of bag-relative path to the manifest(s) used to perform the
           validation
