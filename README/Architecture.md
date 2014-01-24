# Overview

Eventgen a modular set of Python code, structured as a Splunk application, which allows for a number of use cases:

* Stand-alone event generation
* Embedding in Splunk and serving multiple Splunk apps' event generation needs by allowing those apps to specify an eventgen.conf in their Splunk default/local app directories.
* Generation built using sample files and configuration
* Generation using custom code to model a scenario
* Output to Splunk modular input or any number of pluggable output modes

Prior to version 3, Eventgen was a monolithic set of code with dozens of configuration tuneables all driving a complex set of conditional code.  Over its life, Eventgen has grown so many tunables that the code had essentially become unmaintainable.  This drove a desire by Clint Sharp (clint@splunk.com), the primarily Eventgen maintainer, to rearchitect the code to make it more modular.

A second set of requirements also drove the version 3 re-architecture.  There continue to be a number of eventgens written internally to Splunk, and while most of those developers would be happy using Eventgen, either the config syntax is more arduous to learn than writing some python code or the scenario they want to model simply isn't possible using relatively naive methods of looking for events and replacing regex identified tokens with substitutions from random selections.  Replay mode is also insuffucient in that they do not want to have to base the eventgens on real data.

These requirements have led to this new architecture in verison 3.  Eventgen is now completely modular, with the core code base handling a few tasks:

* Configuration management
    * Config can come from Splunk if running as an app or via python's ConfigParser if running standalones
    * Handles flattening of configs from global defaults and individual config files
* Loading Plugins
    * Load plugins from the lib/plugins directory or Apps lib/plugins directories, with three types of Plugins: Rating, Generator and Output
    * For more detail on writing plugins, see the [Plugins documentation](README/Plugins.md)
* Creating timers for each sample
    * For samples with queueable generators (meaning we can multithread generation), the timer wakes up every interval and puts a task in the queue to be generated by a generator worker
    * For samples which are not queueable, the thread runs the generator plugin single threaded and handles sleeping the proper amount to manage intervals
* Managing workers
    * Generator and Output plugins run as separate threads or processes and can have a configurable number of workers running

That's about it, everything else is handed in a plugin.

# Basic Flow

Events get generated through a series of processing queues.  Everything in the core eventgen architecture is multi-threaded.  In order to get around Python's Global Interpreter Lock (GIL), we can also run in a multiprocess mode.  Essentially:

*Timers -> Generator Jobs in Generator Queue -> Output Jobs in Output Queue*

Depending on tunable parameters, these can be threads or processes and can either all run on one machine or multiple machines.

# Scaling

This queue architecture allows to get significant advantage by distributing processing.  At each timer execution a generation job gets put in the queue.  This queue by default is a Python Queue (either in queue.Queue or multiprocessing.Queue), but can also be configured to be a (Zeromq)[http://zeromq.org/] push/pull queue via the network as well.  This is significantly faster than Python's multiprocessing.Queue and allows us to scale beyond one machine.

# Testing

Given the complexity and the reimplementation of a number of features during the version 3 refactor, we've also built out a number of test examples.  We currently don't have test scripts built with assertions etc, but there are a series of configs and samples under the tests/ directory which should demonstrate and allow testing of basic functionality.