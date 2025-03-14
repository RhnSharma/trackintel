Model
*****

In trackintel, **tracking data** is split into several classes. It is not generally 
assumed that data is already available in all these classes, instead, trackintel
provides functionality to generate everything starting from the raw GPS positionfix data 
(consisting of at least ``(user_id, tracked_at, longitude, latitude)`` tuples).

* **users**: The users for which data is available.
* **positionfixes**: Raw GPS data.
* **staypoints**: Locations where a user spent a minimal time.
* **triplegs**: Segments covered with one mode of transport.
* **locations**: Clustered staypoints.
* **trips**: Segments between consecutive activity staypoints (special staypoints that are not just waiting points).
* **tours**: Sequences of trips which start and end at the same location (if the column 'journey' 
  is True, this location is *home*).

An example plot showing the hierarchy of the trackintel data model can be found below:

.. image:: /assets/hierarchy.png
	:align: center
	:height: 390px
	:width: 492px
	:alt: model image

The image below explicitly shows the definition of locations as clustered staypoints, generated by one or several users.

.. image:: /assets/locations_with_pfs.png
	:align: center
	:height: 405px
	:width: 720px 
	:scale: 70
	

A detailed (and SQL-specific) explanation of the different classes can be found under 
:ref:`data_model`.

GeoPandas Implementation
========================

.. highlight:: python

In trackintel, we assume that all these classes are available as (Geo)Pandas (Geo)DataFrames. While we
do not extend the given DataFrame constructs, we provide accessors that validate that a given DataFrame
corresponds to a set of constraints, and make functions available on the DataFrames. For example::

    df = trackintel.read_positionfixes_csv('data.csv')
    df.as_positionfixes.generate_staypoints()

This will read a CSV into a format compatible with the trackintel understanding of a collection of 
positionfixes, and the second line will wrap the DataFrame with an accessor providing functions such 
as ``generate_staypoints()``. You can read up more on Pandas accessors in `the Pandas documentation 
<https://pandas.pydata.org/pandas-docs/stable/development/extending.html>`_.

Available Accessors
===================

The following accessors are available within *trackintel*.

UsersAccessor
-------------

.. autoclass:: trackintel.model.users.UsersAccessor
	:members:

PositionfixesAccessor
---------------------

.. autoclass:: trackintel.model.positionfixes.PositionfixesAccessor
	:members:

StaypointsAccessor
------------------

.. autoclass:: trackintel.model.staypoints.StaypointsAccessor
	:members:

TriplegsAccessor
----------------

.. autoclass:: trackintel.model.triplegs.TriplegsAccessor
	:members:

LocationsAccessor
-----------------

.. autoclass:: trackintel.model.locations.LocationsAccessor
	:members:

TripsAccessor
-------------

.. autoclass:: trackintel.model.trips.TripsAccessor
	:members:

ToursAccessor
-------------

.. autoclass:: trackintel.model.tours.ToursAccessor
	:members:


.. _data_model:

Data Model (SQL)
================


For a general description of the data model, please refer to the 
:doc:`/modules/model`. You can download the 
complete SQL script `here <https://github.com/mie-lab/trackintel/blob/master/sql/create_tables_pg.sql>`__
in case you want to quickly set up a database. Also take a look at the `example on github 
<https://github.com/mie-lab/trackintel/blob/master/examples/setup_example_database.py>`_.

.. highlight:: sql

The **users** table contains information about individual users for which mobility data is available 
(i.e., each ``user_id`` appearing in the tables below should have a corresponding user in the ``users``
table)::

    CREATE TABLE users (
        -- Common to all tables.
        id bigint NOT NULL,

        -- Specific attributes.
        -- The attributes contain additional information that might be given for each user. This
        -- could be demographic information, such as age, gender, or income. 
        attributes json,

        -- Spatial attributes.
        geom_home geometry(Point, 4326),
        geom_work geometry(Point, 4326),

        -- Constraints.
        CONSTRAINT users_pkey PRIMARY KEY (id)
    );

The **positionfixes** table contains all positionfixes (i.e., all individual GPS trackpoints, 
consisting of longitude, latitude and timestamp) of all users. They are not only linked to 
a user, but also (potentially, if this link has already been made) to a tripleg or a staypoint::

    CREATE TABLE positionfixes (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id bigint NOT NULL,

        -- References to foreign tables.
        tripleg_id bigint,
        staypoint_id bigint,

        -- Temporal attributes.
        tracked_at timestamp with time zone NOT NULL,

        -- Specific attributes.
        accuracy double precision,
        tracking_tech character(12),
        -- The context contains additional information that might be filled in by trackintel.
        -- This could include things such as the temperature, public transport stops in vicinity, etc.
        context json,

        -- Spatial attributes.
        elevation double precision,
        geom geometry(Point, 4326),

        -- Constraints.
        CONSTRAINT positionfixes_pkey PRIMARY KEY (id)
    );

The **staypoints** table contains all stay points, i.e., points where a user stayed
for a certain amount of time. They are linked to a user, as well as (potentially) to a trip
and location. Depending on the purpose and time spent, a staypoint can be an *activity*,
i.e., a meaningful destination of movement::

    CREATE TABLE staypoints (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id bigint NOT NULL,

        -- References to foreign tables.
        trip_id bigint,
        location_id bigint,

        -- Temporal attributes.
        started_at timestamp with time zone NOT NULL,
        finished_at timestamp with time zone NOT NULL,
        
        -- Attributes related to the activity performed at the staypoint.
        purpose_detected character varying,
        purpose_validated character varying,
        validated boolean,
        validated_at timestamp with time zone,
        activity boolean,

        -- Specific attributes.
        -- The radius is an approximation of how far the positionfixes that made up this staypoint
        -- are scattered around the center (geom) of it.
        radius double precision,
        -- The context contains additional information that might be filled in by trackintel.
        -- This could include things such as the temperature, public transport stops in vicinity, etc.
        context json,

        -- Spatial attributes.
        elevation double precision,
        geom geometry(Point, 4326),

        -- Constraints.
        CONSTRAINT staypoints_pkey PRIMARY KEY (id)
    );

The **triplegs** table contains all triplegs, i.e., journeys that have been taken 
with a single mode of transport. They are linked to both a user, as well as a trip 
and if applicable, a public transport case::

    CREATE TABLE triplegs (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id bigint NOT NULL,

        -- References to foreign tables.
        trip_id bigint,

        -- Temporal attributes.
        started_at timestamp with time zone NOT NULL,
        finished_at timestamp with time zone NOT NULL,

        -- Attributes related to the transport mode used for this tripleg.
        mode_detected character varying,
        mode_validated character varying,
        validated boolean,
        validated_at timestamp with time zone,

        -- Specific attributes.
        -- The context contains additional information that might be filled in by trackintel.
        -- This could include things such as the temperature, public transport stops in vicinity, etc.
        context json,

        -- Spatial attributes.
        -- The raw geometry is unprocessed, directly made up from the positionfixes. The column
        -- 'geom' contains processed (e.g., smoothened, map matched, etc.) data.
        geom_raw geometry(Linestring, 4326),
        geom geometry(Linestring, 4326),

        -- Constraints.
        CONSTRAINT triplegs_pkey PRIMARY KEY (id)
    );

The **locations** table contains all locations, i.e., somehow created (e.g., from clustering
staypoints) meaningful locations::

    CREATE TABLE locations (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id bigint,

        -- Specific attributes.
        -- The context contains additional information that might be filled in by trackintel.
        -- This could include things such as the temperature, public transport stops in vicinity, etc.
        context json,
        
        -- Spatial attributes.
        elevation double precision,
        extent geometry(Polygon, 4326),
        center geometry(Point, 4326),

        -- Constraints.
        CONSTRAINT locations_pkey PRIMARY KEY (id)
    );

The **trips** table contains all trips, i.e., collection of trip legs going from one 
activity (staypoint with ``activity==True``) to another. They are simply linked to a user::

    CREATE TABLE trips (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id integer NOT NULL,

        -- References to foreign tables.
        origin_staypoint_id bigint,
        destination_staypoint_id bigint,

        -- Temporal attributes.
        started_at timestamp with time zone NOT NULL,
        finished_at timestamp with time zone NOT NULL,
        
        -- Specific attributes.
        -- The context contains additional information that might be filled in by trackintel.
        -- This could include things such as the temperature, public transport stops in vicinity, etc.
        context json,

        -- Constraints.
        CONSTRAINT trips_pkey PRIMARY KEY (id)
    );

The **tours** table contains all tours, i.e., sequence of trips which start and end 
at the same location (in case of ``journey==True`` this location is *home*). 
They are linked to a user::

    CREATE TABLE tours (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id integer NOT NULL,

        -- References to foreign tables.
        origin_destination_location_id bigint,

        -- Temporal attributes.
        started_at timestamp with time zone NOT NULL,
        finished_at timestamp with time zone NOT NULL,
        
        -- Specific attributes.
        journey bool,
        -- The context contains additional information that might be filled in by trackintel.
        -- This could include things such as the temperature, public transport stops in vicinity, etc.
        context json,

        -- Constraints.
        CONSTRAINT tours_pkey PRIMARY KEY (id)
    );
