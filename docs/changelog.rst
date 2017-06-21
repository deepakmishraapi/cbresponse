CbAPI Changelog
===============

CbAPI 1.1.1 - Released June 2, 2017
-----------------------------------

This release includes compatibility fixes for Cb Response 6.1. Changes from 1.0.1 include:

* Substantial changes to the ``Process`` Model Object for Cb Response 6.1. See details below.
* New ``StoragePartition`` Model Object to control Solr core loading/unloading in Cb Response 6.1.
* New ``IngressFilter`` Model Object to control ingress filter settings in Cb Response 6.1.
* Fix issues with ``event_export.py`` example script.
* Add ``.all_events`` property to the ``Process`` Model Object to expose a list of all events across all segments.
* Add example script to perform auto-banning based on watchlist hits from Cb Event Forwarder S3 output files.
* Add bulk operations to the ``ThreatReport`` and ``Alert`` Query objects:
  * You can now call ``.set_ignored()``, ``.assign()``, and ``.change_status()`` on an ``Alert`` Query object to change
    the respective fields for every Alert that matches the query.
  * You can now call ``.set_ignored()`` on a ``ThreatReport`` Query object to set or clear the ignored flag for every
    ThreatReport that matches the query.

Changes to ``Process`` Model Object for Cb Response 6.1
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Cb Response 6.1 uses a new way of recording process events that greatly increases the speed and scale of collection,
allowing you to store and search data for more endpoints on the same hardware. Details on the new database format
can be found on the Developer Network website at the `Process API Changes for Cb Response 6.0
<https://developer.carbonblack.com/reference/enterprise-response/6.1/process-api-changes/>`_ page.

The ``Process`` Model Object traditionally referred to a single "segment" of events in the Cb Response database. In
Cb Response versions prior to 6.0, a single segment will include up to 10,000 individual endpoint events, enough to
handle over 95% of the typical event activity for a given process. Therefore, even though a ``Process`` Model Object
technically refers to a single *segment* in a process, since most processes had less than 10,000 events and therefore
were only comprised of a single segment, this distinction wasn't necessary.

However, now that processes are split across many segments, a better way of handling this is necessary. Therefore,
Cb Response 6.0 introduces the new ``.group_by()`` method. This method is new in cbapi 1.1.0 and is part of five
new query filters available when communicating with a Cb Response 6.1 server. These filters are accessible via methods
on the ``Process`` Query object. These new methods are:

* ``.group_by()`` - Group the result set by a field in the response. Typically you will want to group by ``id``, which
  will ensure that the result set only has one result per *process* rather than one result per *event segment*. For
  more information on processes, process segments, and how segments are stored in Cb Response 6.0, see the
  `Process API Changes for Cb Response 6.0 <https://developer.carbonblack.com/reference/enterprise-response/6.1/process-api-changes/>`_
  page on the Developer Network website.
* ``.min_last_update()`` - Only return processes that have events after a given date/time stamp (relative to the
  individual sensor's clock)
* ``.max_last_update()`` - Only return processes that have events before a given date/time stamp (relative to the
  individual sensor's clock)
* ``.min_last_server_update()`` - Only return processes that have events after a given date/time stamp (relative to the
  Cb Response server's clock)
* ``.max_last_server_update()`` - Only return processes that have events before a given date/time stamp (relative to the
  Cb Response server's clock)

Examples for new Filters
~~~~~~~~~~~~~~~~~~~~~~~~

Let's take a look at an example::

    >>> from datetime import datetime, timedelta
    >>> yesterday = datetime.utcnow() - timedelta(days=1)      # Get "yesterday" in GMT
    >>> for proc in c.select(Process).where("process_name:cmd.exe").min_last_update(yesterday):
    ...     print proc.id, proc.segment
    DEBUG:cbapi.connection:HTTP GET /api/v1/process?cb.min_last_update=2017-05-21T18%3A41%3A58Z&cb.urlver=1&facet=false&q=process_name%3Acmd.exe&rows=100&sort=last_update+desc&start=0 took 2.164s (response 200)
    00000001-0000-0e48-01d2-c2a397f4cfe0 1495465643405
    00000001-0000-0e48-01d2-c2a397f4cfe0 1495465407157
    00000001-0000-0e48-01d2-c2a397f4cfe0 1495463680155
    00000001-0000-0e48-01d2-c2a397f4cfe0 1495463807694
    00000001-0000-0e48-01d2-c2a397f4cfe0 1495463543944
    00000001-0000-0e48-01d2-c2a397f4cfe0 1495463176570
    00000001-0000-0e48-01d2-c2a397f4cfe0 1495463243492

Notice that the "same" process ID is returned seven times, but with seven different segment IDs. Cb Response will
return *every* process event segment that matches a given query, in this case, any event segment that contains the
process command name ``cmd.exe``.

That is, however, most likely not what you wanted. Instead, you'd like a list of the *unique* processes associated with
the command name ``cmd.exe``. Just add the ``.group_by("id")`` filter to your query::

    >>> for proc in c.select(Process).where("process_name:cmd.exe").min_last_update(yesterday).group_by("id"):
    ...     print proc.id, proc.segment
    DEBUG:cbapi.connection:HTTP GET /api/v1/process?cb.group=id&cb.min_last_update=2017-05-21T18%3A41%3A58Z&cb.urlver=1&facet=false&q=process_name%3Acmd.exe&rows=100&sort=last_update+desc&start=0 took 2.163s (response 200)
    00000001-0000-0e48-01d2-c2a397f4cfe0 1495465643405