**********
Quickstart
**********

This guide will quickly introduce you to some of the core features of
pyspotify. It assumes that you've already installed pyspotify. If you do not,
check out :doc:`installation`. For a complete reference of what pyspotify
provides, refer to the :doc:`api`.


Application keys
================

Every app that use libspotify needs its own libspotify application key.
Application keys can be obtained automatically and free of charge from Spotify.

#. Go to the `Spotify developer pages <https://developer.spotify.com/>`__ and
   login using your Spotify account.

#. Find the `libspotify application keys
   <https://developer.spotify.com/technologies/libspotify/keys/>`__ management
   page and request an application key for your application.

#. Once the key is issued, download the "binary" version. The "C code" version
   of the key will not work with pyspotify.

#. If you place the application key in the same directory as your application's
   main Python file, pyspotify will automatically find it and use it. If you
   want to keep the application key in another location, you'll need to set
   :attr:`~spotify.SessionConfig.application_key` or
   :attr:`~spotify.SessionConfig.application_key_filename` in your session
   config.


Creating a session
==================

Once pyspotify is installed and the application key is in place, we can start
writing some Python code. Almost everything in pyspotify requires a
:class:`~spotify.Session`, so we'll start with creating a session with the
default config::

    >>> import spotify
    >>> session = spotify.Session()

All config must be done before the session is created. Thus, if you need to
change any config to something else than the default, you must create a
:class:`~spotify.SessionConfig` object first, and then pass it to the session::

    >>> import spotify
    >>> config = spotify.SessionConfig()
    >>> config.user_agent = 'My awesome Spotify client'
    >>> config.tracefile = b'/tmp/libspotify-trace.log'
    >>> session = spotify.Session(config=config)


Text encoding
=============

libspotify encodes all text as UTF-8. pyspotify converts the UTF-8 bytestrings
to Unicode strings before returning them to you, so you don't have to be
worried about text encoding.

Similarly, pyspotify will convert any string you give it from Unicode to UTF-8
encoded bytestrings before passing them on to libspotify. The only exception is
file system paths, like :attr:`~spotify.SessionConfig.tracefile` above, which
is passed directly to libspotify. This is in case you have a file system which
doesn't use UTF-8 encoding for file names.


Login and event processing
==========================

With a session we can do a few things, like creating objects from Spotify
URIs::

    >>> import spotify
    >>> session = spotify.Session()
    >>> album = spotify.Album('spotify:album:0XHpO9qTpqJJQwa2zFxAAE')
    >>> album
    Album(u'spotify:album:0XHpO9qTpqJJQwa2zFxAAE')
    >>> album.link
    Link(u'spotify:album:0XHpO9qTpqJJQwa2zFxAAE')
    >>> album.link.uri
    u'spotify:album:0XHpO9qTpqJJQwa2zFxAAE'

But that's mostly how far you get with a fresh session. To do more, you need to
login to the Spotify service using a Spotify account with the Premium
subscription. The free ad financed Spotify subscription will not work with
libspotify.

::

    >>> import spotify
    >>> session = spotify.Session()
    >>> session.login('alice', 's3cretpassword')

For alternative ways to login, refer to the :meth:`~spotify.Session.login`
documentation.

The :meth:`~spotify.Session.login` method is asynchronous, so we must ask the
session to :meth:`~spotify.Session.process_events` until the login has
succeeded or failed at which point the
:attr:`~spotify.SessionCallbacks.logged_in` callback will be called::

    >>> session.user is None
    True
    >>> session.process_events()
    >>> session.user
    User(u'spotify:user:alice')

TODO: Waiting for the login to complete should be easier.


Logging
=======

pyspotify uses Python's standard :mod:`logging` module for logging. All log
records emitted by pyspotify are issued to the logger named "spotify", or a
sublogger of it.

Out of the box, pyspotify is set up with :class:`logging.NullHandler` as the
only log record handler. This is the recommended approach for logging in
libraries, so that the application developer using the library will have full
control over how the log records from the library will be exposed to the
application's users. In other words, if you want to see the log records from
pyspotify anywhere, you need to add a useful handler to the root logger or the
logger named "spotify" to get any log output from pyspotify. The defaults
provided by :meth:`logging.basicConfig` is enough to get debug log statements
out of pyspotify::

    import logging
    logging.basicConfig(level=logging.DEBUG)

If your application is already using :mod:`logging`, and you want debug log
output from your own application, but not from pyspotify, you can ignore debug
log messages from pyspotify by increasing the threshold on the "spotify" logger
to "info" level or higher::

    import logging
    logging.basicConfig(level=logging.DEBUG)
    logging.getLogger('spotify').setLevel(logging.INFO)

For more details on how to use :mod:`logging`, please refer to the Python
standard library documentation.

If we turn on logging, the login process is a bit more informative::

    >>> import logging
    >>> logging.basicConfig(level=logging.DEBUG)
    >>> import spotify
    >>> session = spotify.Session()
    >>> session.login('alice', 's3cretpassword')
    DEBUG:spotify.session:Notify main thread
    DEBUG:spotify.session:Log message from Spotify: 19:15:54.829 I [ap:1752] Connecting to AP ap.spotify.com:4070
    DEBUG:spotify.session:Log message from Spotify: 19:15:54.862 I [ap:1226] Connected to AP: 78.31.12.11:4070
    >>> session.process_events()
    DEBUG:spotify.session:Notify main thread
    DEBUG:spotify.session:Log message from Spotify: 19:17:27.972 E [session:926] Not all tracks cached
    INFO:spotify.session:Logged in
    DEBUG:spotify.session:Credentials blob updated: 'NfFEO...'
    DEBUG:spotify.session:Connection state updated
    43
    >>> session.user
    User(u'spotify:user:alice') 


Browsing metadata
=================

When we're logged in, the objects we created from Spotify URIs becomes a lot
more interesting::

    >>> album = spotify.Album('spotify:album:0XHpO9qTpqJJQwa2zFxAAE')

If the object isn't loaded, you can call :meth:`~spotify.Album.load` to block
until the object is loaded with data::

    >>> album.is_loaded
    False
    >>> album.name is None
    True
    >>> album.load()
    Album('spotify:album:0XHpO9qTpqJJQwa2zFxAAE')
    >>> album.name
    u'Reach For Glory'
    >>> album.artist
    Artist(u'spotify:artist:4kjWnaLfIRcLJ1Dy4Wr6tY')
    >>> album.artist.load().name
    u'Blackmill'

The :class:`~spotify.Album` object give you the most basic information about
an album. For more metadata, you can call :meth:`~spotify.Album.browse()` to
get an :class:`~spotify.AlbumBrowser`::

    >>> browser = album.browse()

The browser also needs to load data, but once its loaded, most related objects
are in place with data as well::

    >>> browser.load()
    AlbumBrowser(u'spotify:album:0XHpO9qTpqJJQwa2zFxAAE')
    >>> browser.copyrights
    [u'2011 Blackmill']
    >>> browser.tracks
    [Track(u'spotify:track:4FXj4ZKMO2dSkqiAhV7L8t'),
     Track(u'spotify:track:1sYClIlZZsL6dVMVTxCYRm'),
     Track(u'spotify:track:1uY4O332HuqLIcSSJlg4NX'),
     Track(u'spotify:track:58qbTrCRGyjF9tnjvHDqAD'),
     Track(u'spotify:track:3RZzg8yZs5HaRjQiDiBIsV'),
     Track(u'spotify:track:4jIzCryeLdBgE671gdQ6QD'),
     Track(u'spotify:track:4JNpKcFjVFYIzt1D95dmi0'),
     Track(u'spotify:track:7wAtUSgh6wN5ZmuPRRXHyL'),
     Track(u'spotify:track:7HYOVVLd5XnfY4yyV5Neke'),
     Track(u'spotify:track:2YfVXi6dTux0x8KkWeZdd3'),
     Track(u'spotify:track:6HPKugiH3p0pUJBNgUQoou')]
    >>> [(t.index, t.name, t.duration // 1000) for t in browser.tracks]
    [(1, u'Evil Beauty', 228),
     (2, u'City Lights', 299),
     (3, u'A Reach For Glory', 254),
     (4, u'Relentless', 194),
     (5, u'In The Night Of Wilderness', 327),
     (6, u"Journey's End", 296),
     (7, u'Oh Miah', 333),
     (8, u'Flesh and Bones', 276),
     (9, u'Sacred River', 266),
     (10, u'Rain', 359),
     (11, u'As Time Goes By', 97)]


Downloading cover art
=====================

While we're at it, let's do something a bit more impressive; getting cover
art::

    >>> cover = album.cover(spotify.ImageSize.LARGE)
    >>> cover.load()
    Image(u'spotify:image:16eaba4959d5d97e8c0ca04289e0b1baaefae55f')

Currently, all covers are in JPEG format::

    >>> cover.format
    <ImageFormat.JPEG: 0>

The :class:`~spotify.Image` object gives access to the raw JPEG data::

    >>> len(cover.data)
    37204
    >>> cover.data[:20]
    '\xff\xd8\xff\xe0\x00\x10JFIF\x00\x01\x01\x01\x00H\x00H\x00\x00'

For convenience, it also provides the same data encoded as a ``data:`` URI for
easy embedding into HTML documents::

    >>> len(cover.data_uri)
    49631
    >>> cover.data_uri[:60]
    u'data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEASABIAAD/2wBDAAMCA'

If you're following along, you can try writing the image data out to files and
inspect the result yourself::

    >>> open('/tmp/cover.jpg', 'w+').write(cover.data)
    >>> open('/tmp/cover.html', 'w+').write('<img src="%s">' % cover.data_uri)


Searching
=========

If you don't have the URI to a Spotify object, another way to get started is
to :meth:`~spotify.Session.search`::

    >>> search = session.search('massive attack')
    >>> search.load()
    Search(u'spotify:search:massive+attack')

A search returns lists of matching artists, albums, tracks, and playlists::

    >>> (search.artist_total, search.album_total, search.track_total, track.playlist_total)
    (5, 50, 564, 125)
    >>> search.artists[0].load().name
    u'Massive Attack'
    >>> [a.load().name for a in search.artists[:3]]
    [u'Massive Attack',
     u'Kwanzaa Posse feat. Massive Attack',
     u'Massive Attack Vs. Mad Professor']

Only the first 20 items in each list are returned by default::

    >>> len(search.artists)
    5
    >>> len(search.tracks)
    20

The :class:`~spotify.Search` object can help you with getting
:meth:`~spotify.Search.more` results from the same query::

    >>> search2 = search.more().load()
    >>> len(search2.artists)
    0
    >>> len(search2.tracks)
    20
    >>> search.track_offset
    0
    >>> search.tracks[0]
    Track(u'spotify:track:67Hna13dNDkZvBpTXRIaOJ')
    >>> search2.track_offset
    20
    >>> search2.tracks[0]
    Track(u'spotify:track:3kKVqFF4pv4EXeQe428zl2')

You can also do searches where Spotify tries to figure out what you
mean based on popularity, etc. instead of exact token matches::

    >>> search = session.search('mas').load()
    Search(u'spotify:search:mas')
    >>> search.artists[0].load().name
    u'X-Mas Allstars'

    >>> search = session.search('mas', search_type=spotify.SearchType.SUGGEST).load()
    Search(u'spotify:search:mas')
    >>> search.artists[0].load().name
    u'Massive Attack'


Playlist management
===================

TODO


Playing music
=============

TODO


Thread safety
=============

TODO: Explain that libspotify isn't thread safe. You must either use a single
thread to call pyspotify methods, or protect all pyspotify API usage with a
single lock.
