.. _top-ioda-file-formats:

IODA File Formats
=================

Overview
--------

IODA can read files in the following formats:

* HDF5
* ODB

and write files in the following formats:

* HDF5.

The following sections describe how these formats are handled from the user's point of view.

HDF5
----

To read an HDF5 file into an ``ObsSpace``, it is enough to set the ``obs space.obsdatain.obsfile`` option in the YAML configuration file to the HDF5 file path. For example,

.. code-block:: YAML

    obs space:
      obsdatain:
        obsfile: Data/testinput_tier_1/sondes_obs_2018041500_m.nc4

Similarly, to write the contents of an ``ObsSpace`` to disk at the end of the observation processing pipeline, use the ``obs space.obsdataout.obsfile`` option:

.. code-block:: YAML

    obs space:
      obsdataout:
        obsfile: obsfile: Data/sondes_obs_2018041500_m_out.nc4

Each MPI rank will then write its observations to a separate file with the name obtained by inserting the rank before the extension of the file name taken from the ``obs space.obsdataout.obsfile`` option. In the example above, processes 0 and 1 would produce files called ``Data/sondes_obs_2018041500_m_out_0000.nc4`` and ``Data/sondes_obs_2018041500_m_out_0001.nc4``, respectively (assuming observations were split across ranks only in space; if they were split also in time, file names would have an extra suffix with the index of the time partition).

ODB
---

.. note::

   To be able to read ODB files, ``ioda`` needs to be built in an environment providing access to ECMWF's ``odc`` library. All of the development containers (Intel, GNU and Clang) include this library.

To read an ODB file into an ``ObsSpace``, three options need to be set in the ``obs space.obsdatain`` section of the YAML configuration file:

* ``obsfile``: the path to the ODB file;
* ``mapping file``: the path to a YAML file mapping ODB column names and units to IODA variable names;
* ``query file``: the path to a YAML file defining the parameters of an SQL query selecting the required data from the ODB file.

The syntax of the mapping and query files is described in the subsections below. The ``ioda`` repository contains sample mapping and query files that should be sufficient for most needs. There is a single mapping file, ``test/testinput/odb_default_name_map.yml``, and one query file per observation type, e.g. ``test/testinput/iodatest_odb_aircraft.yml`` for aircraft observations and ``test/testinput/iodatest_odb_atms.yml`` for ATMS observations. For example, a YAML file used for aircraft data processing could contain the following ``obs space.obsdatain`` section:

.. code-block:: YAML

    obs space:
      obsdatain:
        obsfile: Data/testinput_tier_1/aircraft.odb
        mapping file: testinput/odb_default_name_map.yml
        query file: testinput/iodatest_odb_aircraft.yml

Mapping Files
"""""""""""""

Here is an example ODB mapping file:

.. code-block:: YAML

    varno-independent columns:
      - source: lat
        name: MetaData/latitude
      - source: lon
        name: MetaData/longitude
      - source: level.surface
        name: MetaData/surface_level
        bit index: 0
      - source: level.tropopause_level
        name: MetaData/tropopause_level
        bit index: 2
    varno-dependent columns:
      - source: initial_obsvalue
        group name: ObsValue
        varno-to-variable-name mapping: &obsvalue_varnos
          - varno: 29
            name: relative_humidity
            unit: percentage
          - varno: 110
            name: surface_pressure
            unit: hectopascal
      - source: initial_obsvalue
        group name: MetaData
        varno-to-variable-name mapping:
          - varno: 235
            name: air_pressure
      - source: obs_error
        group name: ObsError
        varno-to-variable-name mapping: *obsvalue_varnos
      - source: datum_event1.duplicate
        group name: DiagnosticFlags/Duplicate
        bit index: 17
        varno-to-variable-name mapping:
          - varno: 29
            name: relative_humidity
          - varno: 110
            name: surface_pressure
    complementary variables:
      - input names: [site_name_1, site_name_2, site_name_3, site_name_4]
        output name: MetaData/station_id

A mapping file may contain up to three top-level sections: ``varno-independent columns``, ``varno-dependent columns`` and ``complementary variables``. All of them are optional, but at least the first two will typically be present. The syntax of each section is described below, followed by a detailed explanation of the mappings defined in the above YAML file.

The ``varno-independent columns`` Section
.........................................

This section contains a list of items defining the mapping of individual varno-independent ODB columns to ``ioda`` variables. Varno-independent columns are those storing values dependent on the observation location, but not on the observed variable (identified by its *varno*). They include most metadata, such as latitude, longitude or station ID. Each item in this list may contain the following keys:

* ``source`` (required): name of the mapped ODB column (e.g. ``lat``) or a member of a bitfield column (e.g. ``level.surface``, indicating the ``surface`` member of the ``level`` column of type *bitfield*).

* ``name`` (required): full name of the corresponding ``ioda`` variable (e.g. ``MetaData/latitude``);

.. _varno-independent columns.unit:

* ``unit`` (optional): name of the unit used in the ODB file. If specified, values loaded from the ODB file will be converted to the unit used in ``ioda`` (typically a basic SI unit). Currently the following units are supported: ``celsius``, ``knot``, ``percentage`` (converted to a fraction), ``okta`` (1/8 -- converted to a fraction), ``degree`` (converted to radians) and ``hectopascal`` (converted to pascals).

* ``bit index`` (optional): 0-based index of the bit within a bitfield column that should store the values of the mapped member. Will be used by the ODB file writer, currently in development.

.. note::

   Bitfield ODB columns can either be mapped in their entirety to a single integer ``ioda`` variable  or be split into multiple Boolean ``ioda`` variables, each storing the value of a single member. In the latter case, it is not necessary to map each member to a ``ioda`` variable: some may be omitted, as illustrated for the ``level`` column in the YAML snippet above, which contains no mapping for the ``standard_level`` member stored in bit 1.

The ``varno-dependent columns`` Section
.......................................

This section contains a list of items defining the mapping of individual varno-dependent ODB columns to groups of ``ioda`` variables. Varno-dependent columns are those storing values dependent not only on the observation location, but also on the observed variable (identified by its *varno*). Typical examples are the columns storing the observed value or estimated observation error. Each item in this list may contain the following keys:

* ``source`` (required): name of the mapped ODB column (e.g. ``initial_obsvalue``) or a member of a bitfield column (e.g. ``datum_event1.duplicate``, indicating the ``duplicate`` member of the ``datum_event1`` column of type *bitfield*);

* ``group name`` (required): name of the group (e.g. ``ObsValue``) containing the ``ioda`` variables storing restrictions of the mapped ODB column to individual *varnos*;

* ``bit index`` (optional): 0-based index of the bit within a bitfield column that should store the values of the mapped member. Will be used by the ODB file writer, currently in development.

* ``varno-to-variable-name mapping`` (required): a list of items defining the mapping between varnos and ``ioda`` variables. Each item in the list may contain the following keys:

  - ``varno`` (required): numeric identifier of a geophysical variable (see https://apps.ecmwf.int/odbgov/varno for the full list);

  - ``name`` (required) name of the corresponding ``ioda`` variable;

  - ``unit`` (optional): name of the unit used in the ODB file; see :ref:`above <varno-independent columns.unit>` for more details.

The ``complementary variables`` section
............................................

This section contains a list of items defining groups of varno-independent ODB text columns that should be merged into single ``ioda`` variables. This merging is required because entries of ODB text columns are limited to 8 characters each. Within each item, the following keys are recognized:

* ``input names`` (required): ordered list of names of ODB columns that should be merged;
* ``output name`` (required): name of the ``ioda`` variable that will hold the contents of the merged columns;
* ``output variable data type`` (optional): if present, must be set to ``string``;
* ``merge method`` (optional): if present, must be set to ``concat``.

Example Mapping File: Detailed Discussion
.........................................

The example YAML file shown above defines the following mappings:

* The ``lat`` and ``lon`` ODB columns are mapped to the ``MetaData/latitude`` and ``MetaData/longitude`` ``ioda`` variables, respectively. For each column, the value of only one row per location is transferred to the corresponding ``ioda`` variable. (The columns are declared to be varno-independent, so by definition it should not matter which of these rows is used.)

* The ``surface`` and ``tropopause_level`` members of the ``level`` bitfield column are mapped to the ``MetaData/surface_level`` and ``MetaData/tropopause_level`` Boolean ``ioda`` variables, respectively. In each case, the value of only one row per location is transferred to the corresponding ``ioda`` variable.

* Elements of the ``initial_obsvalue`` column located in rows storing observations of varnos 29 and 110 are transferred to the ``ObsValue/relative_humidity`` and ``ObsValue/surface_pressure`` ``ioda`` variables. In each case, a unit conversion takes place.

* Elements of the ``initial_obsvalue`` column located in rows storing observations of varno 235 are transferred to the ``MetaData/air_pressure`` ``ioda`` variable.

* Elements of the ``obs_error`` column located in rows storing observations of varnos 29 and 110 are transferred to the ``ObsError/relative_humidity`` and ``ObsError/surface_pressure`` ``ioda`` variables. In each case, a unit conversion takes place.

* Elements of the ``duplicate`` member of the ``datum_event1`` bitfield column located in rows storing observations of varnos 29 and 110 are transferred to the ``DiagnosticFlags/Duplicate/relative_humidity`` and ``DiagnosticFlags/Duplicate/surface_pressure`` Boolean ``ioda`` variables.

* Strings from the ``site_name_1``, ``site_name_2``, ``site_name_3`` and ``site_name_4`` columns are concatenated and transferred to the ``MetaData/station_id`` ``ioda`` variable. Only one row per location is kept.

.. note::

   Certain variables are handled in a special way.  Columns for date and time (``date``, ``time``, ``receipt_date``, ``receipt_time``) are not specified in the mapping file; instead they are converted into the string date/time representations used by ``ioda`` and stored in ``ioda`` variables ``MetaData/datetime`` and ``MetaData/receiptdatetime``.  They still need to be provided in the ``variables`` list in the query file.

Query files
"""""""""""

The following ODB query file

.. code-block:: YAML

    variables:
    - name: date
    - name: time
    - name: receipt_date
    - name: receipt_time
    - name: lat
    - name: lon
    - name: flight_phase
    - name: level.surface_level
    - name: initial_obsvalue
    where:
      varno: [2,111,112]

corresponds to the following SQL query:

.. code-block:: SQL

    SELECT date, time, receipt_date, receipt_time, lat, lon, flight_phase, initial_obsvalue, level.surface_level
    FROM <ODB file name> 
    WHERE (varno = 2 OR varno = 111 OR varno = 112);

This is the query used to retrieve data from the input ODB file. The names of the specified columns are converted to ``ioda`` variable names when the ObsSpace object is constructed.

In general, a query file must contain a ``where`` section with the ``varno`` key set to the list of identifiers of the geophysical variables of interest (see https://apps.ecmwf.int/odbgov/varno for the full list). In addition, it can contain an optional ``variables`` list; the ``name`` key in each item in this list is the name of a column or a bitfield column member to be retrieved from the ODB file. If the mapping file defines mappings for individual members of a bitfield column and the ``variables`` list contains just the name of this column (rather than names of specific members), all members for which mappings exist are retrieved. Finally, an optional ``ignored names`` key can be set to a list of names of ODB columns that should not be mapped to ``ioda`` variables according to the rules defined in the mapping file even if they are loaded from the ODB file for other reasons. By default, this applies to the following columns: ``date``, ``time``, ``receipt_date``, ``receipt_time``, ``entryno``, ``seqno``, ``varno``, ``vertco_type`` and ``ops_obsgroup``.
