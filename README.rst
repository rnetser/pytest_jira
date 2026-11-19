Intro
=====

A `pytest <http://pytest.org/latest/>`__ plugin for JIRA integration.

This plugin links tests with JIRA tickets. The plugin behaves similar to
the `pytest-bugzilla <https://pypi.python.org/pypi/pytest-bugzilla>`__
plugin.

The plugin does not close JIRA tickets, or create them. It just allows
you to link tests to existing tickets.

Please feel free to contribute by forking and submitting pull requests
or by submitting feature requests or issues to
`issues <https://github.com/rhevm-qe-automation/pytest_jira/issues>`__.

Test results
------------
-  If the test **unresolved** ...

   -  and the *run=False*, the test is **skipped**

   -  and the *run=True* or not set, the test is executed and based on it
      the result is **xpassed** (e.g. unexpected pass) or **xfailed** (e.g. expected fail).
      Interpretation of **xpassed** result depends on the py.test ini-file **xfail_strict** value,
      i.e. with *xfail_strict=true* **xpassed** results will fail the test suite.
      More information about strict xfail available on the py.test `doc <https://docs.pytest.org/en/latest/skipping.html#strict-parameter>`__

-  If the test **resolved** ...

   -  the test is executed and based on it
      the result is **passed** or **failed**

- If the **skipif** parameter is provided ...

  -  with value *False* or *callable returning False-like value* jira marker line is **ignored**


**NOTE:** You can set default value for ``run`` parameter globally in config
file (option ``run_test_case``) or from CLI
``--jira-do-not-run-test-case``. Default value is ``run=True``.

Marking tests
-------------
You can specify jira issue ID in docstring or in pytest.mark.jira decorator.

By default the regular expression pattern for matching jira issue ID is ``[A-Z]+-[0-9]+``,
it can be changed by ``--jira-issue-regex=REGEX`` or in a config file by
``jira_regex=REGEX``.

It's also possible to change behavior if issue ID was not found
by setting ``--jira-marker-strategy=STRATEGY`` or in config file
as ``marker_strategy=STRATEGY``.

Strategies for dealing with issue IDs that were not found:

- **open** - issue is considered as open (default)
- **strict** - raise an exception
- **ignore** - issue id is ignored
- **warn** - write error message and ignore

Issue ID in decorator
~~~~~~~~~~~~~~~~~~~~~
If you use decorator you can specify optional parameters ``run`` and ``skipif``.
If ``run`` is false and issue is unresolved, the test will be skipped.
If ``skipif`` is is false jira marker line will be ignored.

.. code:: python

  @pytest.mark.jira("ORG-1382", run=False)
  def test_skip(): # will be skipped if unresolved
      assert False

  @pytest.mark.jira("ORG-1382")
  def test_xfail(): # will run and xfail if unresolved
      assert False

  @pytest.mark.jira("ORG-1382", skipif=False)
  def test_fail():  # will run and fail as jira marker is ignored
      assert False

Using lambda value for skipif
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
You can use lambda value for ``skipif`` parameter. Lambda function must take
issue JSON as input value and return boolean-like value. If any JIRA ID
gets False-like value marker for that issue will be ignored.

.. code:: python

  @pytest.mark.jira("ORG-1382", skipif=lambda i: 'my component' in i['components'])
  def test_fail():  # Test will run if 'my component' is not present in Jira issue's components
      assert False

  @pytest.mark.jira("ORG-1382", "ORG-1412", skipif=lambda i: 'to do' == i['status'])
  def test_fail():  # Test will run if either of JIRA issue's status differs from 'to do'
      assert False


Issue ID in docstring
~~~~~~~~~~~~~~~~~~~~~

You can disable searching for issue ID in doc string by using
``--jira-disable-docs-search`` parameter or by ``docs_search=False``
in ``jira.cfg``.

.. code:: python

  def test_xpass(): # will run and xpass if unresolved
  """issue: ORG-1382"""
      assert True

Status evaluation
-----------------

Issues are considered as **resolved** if their status matches
``resolved_statuses``. By default it is ``Resolved`` or ``Closed``.

You can set your own custom resolved statuses on command line
``--jira-resolved-statuses``, or in config file.

If you specify components (in command line or jira.cfg), open issues will be considered
**unresolved** only if they are also open for at least one used component.

If you specify version, open issues will be **unresolved** only if they also affects your version.
Even when the issue is closed, but your version was affected and it was not fixed for your version,
the issue will be considered **unresolved**.

If you specify fixed resolutions closed issues will be **unresolved** if they do not also have a **resolved** resolution.

Fixture usage
-------------

Besides a test marker, you can also use the added ``jira_issue`` fixture. This enables examining issue status mid test
and not just at the beginning of a test. The fixture returns a boolean representing the state of the issue.
If the issue isn't found, or the jira plugin isn't loaded, it returns ``None``.

The same API is available at other pytest scopes via ``jira_issue_scope_class``,
``jira_issue_scope_module``, ``jira_issue_scope_package``, and ``jira_issue_scope_session``.

.. code:: python

    NICE_ANIMALS = ["bird", "cat", "dog"]

    def test_stuff(jira_issue):
        animals = ["dog", "cat"]
        for animal in animals:
            if animal == "dog" and jira_issue("ORG-1382") is True:
                print("Issue is still open, cannot check for dogs!")
                continue
            assert animal in NICE_ANIMALS

Requires
========

-  pytest >= 2.2.3
-  requests >= 2.13.0
-  six
-  retry2>=0.9.5
-  marshmallow>=3.2.0

Installation
============

``pip install pytest_jira``

Usage
=====


1. From a source checkout, copy the shipped ``jira.cfg.example`` to
   ``jira.cfg`` (a pip installed package does not ship the file - see the
   template in the next block instead):

   .. code:: sh

     cp jira.cfg.example jira.cfg

   Then put your ``jira.cfg`` at least in one of following places. It is read
   from all of them, later files overriding earlier ones.

   * /etc/jira.cfg
   * tests\_root\_dir/jira.cfg
   * ~/jira.cfg
   * ./jira.cfg (current working directory)

   ``jira.cfg`` is git ignored, so credentials stay out of version control;
   ``jira.cfg.example`` is the template and is safe to commit.

   See example bellow, you can use it as template, and update it according
   to your needs (this is the content of ``jira.cfg.example``).

   .. code:: ini

     [DEFAULT]
     url = https://jira.atlassian.com
     # Leave the credential fields empty for anonymous access.
     username =
     password =
     token =
     cloud_id =
     oauth_client_id =
     oauth_client_secret =
     # ssl_verification = True/False
     # version = foo-1.0
     # components = com1,second component,com3
     # marker_strategy = [open|strict|warn|ignore] (not found issues)
     # docs_search = False (disable searching for issue id in docs)
     # issue_regex = REGEX (replace default `[A-Z]+-[0-9]+` regular expression)
     # resolved_statuses = comma separated list of statuses (closed, resolved)
     # resolved_resolutions = comma separated list of resolutions (done, fixed)
     # run_test_case = True (default value for 'run' parameter)
     # error_strategy = [strict|skip|ignore] (how to handle connection errors)
     # return_jira_metadata = False (return Jira issue with metadata instead of boolean result)
     # connection_retry_total = 5 (number of retries)
     # connection_retry_backoff_factor = 0.2 (connection retry backoff factor)

   Alternatively, every connection field can come from an environment
   variable:

    .. code:: sh

      export PYTEST_JIRA_URL="https://..."
      export PYTEST_JIRA_PASSWORD="FOO"
      export PYTEST_JIRA_USERNAME="BAR"
      export PYTEST_JIRA_TOKEN="TOKEN"
      export PYTEST_JIRA_CLOUD_ID="CLOUD_ID"
      export PYTEST_JIRA_OAUTH_CLIENT_ID="CLIENT_ID"
      export PYTEST_JIRA_OAUTH_CLIENT_SECRET="CLIENT_SECRET"

   The ``PYTEST_JIRA_*`` environment variables win over both ``jira.cfg``
   and the command line options; every other option can also be given on
   the command line.
   For all available command line options run following command.

   .. code:: sh

     py.test --help

2. Mark your tests with jira marker and issue id.

   ``@pytest.mark.jira('issue_id')``

   You can put Jira ID into doc string of test case as well.

3. Run py.test with jira option to enable the plugin.

   ``py.test --jira``

Authentication (Jira Cloud and Server/DC)
==========================================

There are exactly two ways to authenticate against Jira Cloud, plus the
Server/Data Center personal access token:

.. list-table::
   :header-rows: 1

   * - Who
     - Configure
     - Wire
   * - User (Jira Cloud)
     - ``url`` + ``username`` (your email) + ``token`` (API token)
     - basic auth to the site URL
   * - Service account (Jira Cloud)
     - ``url`` + ``cloud_id`` + ``oauth_client_id`` + ``oauth_client_secret``
     - OAuth 2.0 access token, Bearer to the API gateway
   * - Jira Server/Data Center
     - ``url`` + ``token`` (personal access token)
     - Bearer to the URL as configured

A **token on its own is not a valid Jira Cloud configuration** and the plugin
stops with an explanation instead of sending it: a Cloud token belongs either
to a user (then it needs the email as the username) or to a service account
(then it needs the OAuth 2.0 credential).

User (Jira Cloud)
-----------------

.. code:: sh

  export PYTEST_JIRA_URL="https://your-site.atlassian.net"
  export PYTEST_JIRA_USERNAME="you@example.com"
  export PYTEST_JIRA_TOKEN="<API token>"

  py.test --jira

Service account (Jira Cloud)
----------------------------

A service account uses an *OAuth 2.0 credential*: the client id and the
permanent client secret. The plugin exchanges them at
``https://auth.atlassian.com/oauth/token`` (``client_credentials``) for a
short-lived access token, caches it until shortly before it expires, and
sends it as a Bearer token to
``https://api.atlassian.com/ex/jira/<cloudId>``. The client secret never
reaches the API gateway.

.. code:: sh

  export PYTEST_JIRA_URL="https://your-site.atlassian.net"
  export PYTEST_JIRA_CLOUD_ID="<cloudId>"
  export PYTEST_JIRA_OAUTH_CLIENT_ID="<client id>"
  export PYTEST_JIRA_OAUTH_CLIENT_SECRET="<client secret>"

  py.test --jira

With the ``jira.cfg`` keys or command line options instead:
``--jira-oauth-client-id`` / ``oauth_client_id`` and
``--jira-oauth-client-secret`` / ``oauth_client_secret``. Prefer the
environment variable or ``jira.cfg`` for the secret: a command line value is
visible in the process list.

Notes:

* Do not set a username: the minted access token is sent as a Bearer token.
  A username (or a token) configured next to the OAuth credential wins and
  the OAuth credential is then ignored - the plugin says so on stderr. Use
  exactly one of the two ways.
* ``cloud_id`` is always required: the plugin derives the gateway from it.
  ``url`` stays the site URL and is only used for browse links
  (``https://<site>.atlassian.net/browse/<issue>``); pointing ``url`` at
  ``https://api.atlassian.com/ex/jira/<cloudId>`` is accepted but then the
  skip/xfail reasons cannot link to an issue page (a warning says so).
* A failed mint stops the run with the HTTP status and which value to check
  under the default ``error_strategy=strict``; with ``skip``/``ignore`` the
  tests skip and the reason is printed on stderr. The client secret is never
  included in messages.
* Scopes are not permissions: the service account must also have product
  access to Jira and *Browse projects* on every project you mark tests with,
  or Jira answers ``404`` and the issue looks missing to the plugin.

Jira Server/Data Center
-----------------------

.. code:: sh

  export PYTEST_JIRA_URL="https://jira.example.com"
  export PYTEST_JIRA_TOKEN="<personal access token>"

  py.test --jira

A Bearer token is used as configured; no cloud id, no OAuth credential.
Server/Data Center also accepts ``username`` + ``password``
(``--jira-user`` / ``--jira-password``) for basic auth. Jira Cloud has
rejected password basic auth since 2019: use an API token.

Failure behaviour
-----------------

A configuration the plugin cannot use aborts the run with exit code 4
(``USAGE_ERROR``) and one clear message instead of a traceback:

* a token on a Jira Cloud host without a username or an OAuth credential,
* an OAuth credential with a missing half or without a ``cloud_id``,
* a minted access token rejected by the API gateway,
* credentials rejected by the authentication probe (``/rest/api/2/myself``)
  under the default ``error_strategy=strict``; with ``skip``/``ignore`` the
  tests skip and the hint is printed on stderr.

Four combinations are answerable rather than fatal, and are reported on
stderr: an OAuth credential ignored because a username with a token/password
is set, a token ignored because an OAuth credential is set, a ``cloud_id``
ignored because no OAuth service account is configured, and a gateway ``url``
that cannot produce browsable links.

Errors on individual issues (a 4xx after a successful connection) keep the
normal ``--jira-connection-error-strategy`` behaviour.

Common notes
------------

* Values read from environment variables may be quoted
  (``PYTEST_JIRA_URL="https://..."``); a matching pair of surrounding quotes
  and any whitespace are stripped.
* ``PYTEST_JIRA_*`` environment variables win over ``jira.cfg`` *and* the
  command line options.
* Running the plugin's own test suite: set either ``TEST_JIRA_USER`` and
  ``TEST_JIRA_TOKEN`` (user), or ``TEST_JIRA_OAUTH_CLIENT_ID``,
  ``TEST_JIRA_OAUTH_CLIENT_SECRET`` and ``TEST_JIRA_CLOUD_ID`` (service
  account); ``TEST_JIRA_URL`` selects the site. All of them are in
  ``tox.ini``'s ``passenv``; without them the corresponding tests skip. The
  CI workflow carries only ``TEST_JIRA_TOKEN``/``TEST_JIRA_USER``/
  ``TEST_JIRA_URL``/``TEST_JIRA_CLOUD_ID``, so the OAuth path is exercised
  locally, not in CI.


Tests
=====

In order to execute tests run

.. code:: sh

  $ tox
