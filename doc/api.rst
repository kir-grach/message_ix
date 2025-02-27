Python API
==========

The application programming interface (API) for |MESSAGEix| model developers is implemented in Python.
The full API is also available from R; see :doc:`rmessageix`.

.. contents::
   :local:


``ixmp`` package
----------------

:mod:`ixmp` provides three classes. These are fully described by the :doc:`ixmp documentation <ixmp:index>`, which is cross-linked from many places in the |MESSAGEix| documentation.

.. autosummary::

   ~ixmp.Platform
   ~ixmp.TimeSeries
   ~ixmp.Scenario

:mod:`ixmp` also provides some utility classes and methods:

.. autosummary::

   ixmp.config
   ixmp.model.MODELS
   ixmp.model.get_model
   ixmp.testing.make_dantzig


.. currentmodule:: message_ix

``message_ix`` package
----------------------

|MESSAGEix| models are created using the :py:class:`message_ix.Scenario` class. Several utility methods are also provided in the module :py:mod:`message_ix.utils`.

.. autoclass:: message_ix.Scenario
   :members:
   :exclude-members: add_macro
   :show-inheritance:
   :inherited-members:

   This class extends :class:`ixmp.Scenario` and :class:`ixmp.TimeSeries` and
   inherits all their methods. Documentation of these inherited methods is
   included here for convenience. :class:`message_ix.Scenario` defines
   additional methods specific to |MESSAGEix|:

   .. versionchanged:: 3.0

      :meth:`.read_excel` and :meth:`.to_excel` are now methods of :class:`ixmp.Scenario`, but continue to work with message_ix.Scenario.

   .. autosummary::

      add_cat
      add_horizon
      add_macro
      add_spatial_sets
      cat
      cat_list
      equ
      firstmodelyear
      par
      read_excel
      rename
      to_excel
      var
      vintage_and_active_years
      years_active

   .. automethod:: add_macro

      .. warning:: MACRO support via :meth:`add_macro` is **experimental** in message_ix 3.0 and may not function as expected on all possible |MESSAGEix| models.
         See `a list of known and pending issues <https://github.com/iiasa/message_ix/issues?q=is%3Aissue+is%3Aopen+label%3Amacro>`_ on GitHub.


Model classes
-------------

.. currentmodule:: message_ix.models

.. autosummary::

   MESSAGE
   MACRO
   MESSAGE_MACRO
   GAMSModel
   DEFAULT_CPLEX_OPTIONS
   MESSAGE_ITEMS

.. autodata:: DEFAULT_CPLEX_OPTIONS

   These configure the GAMS CPLEX solver (or another solver, if selected); see `the solver documentation <https://www.gams.com/latest/docs/S_CPLEX.html>`_ for possible values.

.. autoclass:: GAMSModel
   :members:
   :exclude-members: defaults

   The :class:`.MESSAGE`, :class:`MACRO`, and :class:`MESSAGE_MACRO` child classes encapsulate the GAMS code for the core MESSAGE (or MACRO) mathematical formulation.

   The class receives `model_options` via :meth:`.Scenario.solve`. Some of these are passed on to the parent class :class:`ixmp.model.gams.GAMSModel` (see there for a list); others are handled as described below.

   The “model_dir” option may be set in the user's :ref:`ixmp configuration file <ixmp:configuration>` using the key “message model dir”.
   If not set, it defaults to “message_ix/model” below the directory where :mod:`message_ix` is installed.

   The “solve_options” option may be set in the user's ixmp configuration file using the key “message solve options”.
   If not set, it defaults to :data:`.DEFAULT_CPLEX_OPTIONS`.

   For example, with the following configuration file:

   .. code-block:: yaml

      {
        "platform": {
          "default": "my-platform",
          "my-platform": {"backend": "jdbc", "etc": "etc"},
        },
        "message model dir": "/path/to/custom/gams/source/files",
        "message solve options": {"lpmethod": 4},
      }

   The following are equivalent:

   .. code-block:: python

      # Model options given explicitly
      scen.solve(
          model_dir="/path/to/custom/gams/source/files",
          solve_options=dict(lpmethod=4),
      )

      # Model options are read from configuration file
      scen.solve()

   The following tables list all model options:

   .. list-table:: Options in :class:`message_ix.models.GAMSModel` or overridden from :mod:`ixmp`
      :widths: 20 40 40
      :header-rows: 1

      * - Option
        - Usage
        - Default value
      * - **model_dir**
        - Path to GAMS source files.
        - See above.
      * - **model_file**
        - Path to GAMS source file.
        - ``'{model_dir}/{model_name}_run.gms'``
      * - **in_file**
        - Path to write GDX input file.
        - ``'{model_dir}/data/MsgData_{case}.gdx'``
      * - **out_file**
        - Path to read GDX output file.
        - ``'{model_dir}/output/MsgOutput_{case}.gdx'``
      * - **solve_args**
        - Arguments passed directly to GAMS.
        - .. code-block::

             [
                 '--in="{in_file}"',
                 '--out="{out_file}"',
                 '--iter="{model_dir}/output/MsgIterationReport_{case}.gdx"'
             ]
      * - **solve_options**
        - Options for the GAMS LP solver.
        - :data:`.DEFAULT_CPLEX_OPTIONS`

   .. list-table:: Option defaults inherited from :class:`ixmp.model.gams.GAMSModel`
      :widths: 20 80
      :header-rows: 1

      * - Option
        - Default value
      * - **case**
        - ``'{scenario.model}_{scenario.scenario}'``
      * - **gams_args**
        - ``['LogOption=4']``
      * - **check_solution**
        - :obj:`True`
      * - **comment**
        - :obj:`None`
      * - **equ_list**
        - :obj:`None`
      * - **var_list**
        - :obj:`None`

.. autoclass:: MESSAGE
   :members: initialize
   :exclude-members: defaults
   :show-inheritance:

   .. autoattribute:: name

.. autoclass:: MACRO
   :members:
   :show-inheritance:

   .. autoattribute:: name

.. autoclass:: MESSAGE_MACRO
   :members:
   :show-inheritance:

   MESSAGE_MACRO solves the MESSAGE and MACRO models iteratively, connecting changes in technology activity and resource demands (from MESSAGE) to changes in final demands and prices (from MACRO).
   This iteration continues until the solution *converges*; i.e. the two models reach a stable point for the values of these parameters.

   MESSAGE_MACRO accepts three additional *model_options* that control the behaviour of this iteration algorithm:

   - **max_adjustment** (:class:`float`, default 0.2): the maximum absolute relative change in final demands between iterations.
     If MACRO returns demands that have changed by more than a factor outside the range (1 - `max_adjustment`, 1 + `max_adjustment`) since the previous iteration, then the change is confined to the limits of that range for the next run of MESSAGE.
   - **convergence_criterion** (:class:`float`, default 0.01): threshold for model convergence.
     This option applies to the same value as `max_adjustment`: the relative change in final demands between two iterations.
     If the absolute relative change is less than `convergence_criterion`, the linked model run is complete.
   - **max_iteration** (:class:`int`, default 50): the maximum number of iterations between the two models.
     If the solution does not converge after this many iterations, the linked model run fails and no valid result is produced.

   .. seealso:: :meth:`.Scenario.add_macro`

   .. autoattribute:: name

.. autodata:: MESSAGE_ITEMS
   :annotation: = dict(…)

   Keys are the names of items (sets, parameters, variables, and equations); values are :class:`dict` specifying their type and dimensionality, with keys 'ix_type', 'idx_sets', and in some cases 'idx_names'.
   These include all items listed in the MESSAGE mathematical specification, i.e. :ref:`sets_maps_def` and :ref:`parameter_def`.

   .. seealso:: :meth:`.MESSAGE.initialize`, :data:`.MACRO_ITEMS`

.. currentmodule:: message_ix.macro

.. autodata:: MACRO_ITEMS
   :annotation: = dict(…)

   All of the items in the MACRO mathematical formulation.

   .. seealso:: :data:`.MESSAGE_ITEMS`


.. _utils:

Utility methods
---------------

.. automodule:: message_ix.util
   :members: make_df


Testing utilities
-----------------

.. automodule:: message_ix.testing
   :members: make_dantzig, make_westeros
