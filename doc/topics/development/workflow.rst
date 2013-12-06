=============
Salt workflow
=============

This document explains a high-level workflow: how Salt-cli understands what to do,
how Salt-master sends a command to a Salt-minion, how Salt-minion takes and run this
command, and how Salt-minion returns a result to the Salt-master.

Let's take a look on what's going on when you run :code:`/usr/bin/salt \* test.ping`.

===
CLI
===

``/usr/bin/salt`` actually lives in ``salt/scripts.py``:

.. code-block:: python

    def salt_main():
        '''
        Publish commands to the salt system from the command line on the
        master.
        '''
        if '' in sys.path:
            sys.path.remove('')
        try:
            client = salt.cli.SaltCMD()
            client.run()
        except KeyboardInterrupt:
            raise SystemExit('\nExiting gracefully on Ctrl-c')

Interesting thing here is :py:``salt.cli.SaltCMD`` and :py:``run`` function.

=======
SaltCMD
=======

You can find this class in ``salt/cli/__init__.py``.
Let's see what's going on in :py:``run`` function.

.. code-block:: python

    def run(self):
        '''
        Execute the salt command line
        '''
        # Skip unnecessarry lines
        try:
            local = salt.client.LocalClient(self.get_config_file_path())
        except SaltClientError as exc:
            self.exit(2, '{0}\n'.format(exc))
            return
        # Skip those too
        try:
            if local:
                # Again, skip some unimportant lines
                cmd_func = local.cmd_cli
                for full_ret in cmd_func(**kwargs):
                    ret, out = self._format_ret(full_ret)
                    self._output_ret(ret, out)
        except (SaltInvocationError, EauthAuthenticationError) as exc:
            ret = str(exc)
            out = ''
            self._output_ret(ret, out)

It's very simple - we create ``LocalClient`` instance, and then run
``cmd_cli`` method and print it's result.

Let's go deeper.

===================
LocalClient.cmd_cli
===================

This method (and LocalClient class itself) lives in ``salt/client/__init__.py``.

.. code-block:: python

    def cmd_cli(
            self,
            tgt,
            fun,
            arg=(),
            timeout=None,
            expr_form='glob',
            ret='',
            verbose=False,
            kwarg=None,
            **kwargs):
        '''
        Used by the :command:`salt` CLI. This method returns minion returns as
        the come back and attempts to block until all minions return.

        The function signature is the same as :py:meth:`cmd` with the
        following exceptions.

        :param verbose: Print extra information about the running command
        :returns: A generator
        '''
        arg = condition_kwarg(arg, kwarg)
        pub_data = self.run_job(
            tgt,
            fun,
            arg,
            expr_form,
            ret,
            timeout,
            **kwargs)

        if not pub_data:
            yield pub_data
        else:
            try:
                for fn_ret in self.get_cli_event_returns(
                        pub_data['jid'],
                        pub_data['minions'],
                        self._get_timeout(timeout),
                        tgt,
                        expr_form,
                        verbose,
                        **kwargs):

                    if not fn_ret:
                        continue

                    yield fn_ret
            except KeyboardInterrupt:
                msg = ('Exiting on Ctrl-C\nThis job\'s jid is:\n{0}\n'
                       'The minions may not have all finished running and any '
                       'remaining minions will return upon completion. To '
                       'look up the return data for this job later run:\n'
                       'salt-run jobs.lookup_jid {0}').format(pub_data['jid'])
                raise SystemExit(msg)

In :py:``self.run_job`` method salt sends a command to a minion, and takes back a result by running
:py:``self.get_cli_event_returns`` method.
