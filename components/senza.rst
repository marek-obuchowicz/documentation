.. _senza:

=====
Senza
=====

**Senza** is STUPS' deployment tool to create and execute `AWS CloudFormation templates`_ in a sane way.

See the :ref:`deployment` section for details on how to deploy applications using Senza, :ref:`pierone` and :ref:`taupage`.

Installation
============

Install or upgrade to the latest version of Senza (|senza-pypi-version|) with PIP:

.. |senza-pypi-version| image:: https://img.shields.io/pypi/v/stups-senza.svg
   :target: https://pypi.python.org/pypi/stups-senza/
   :alt: Latest PyPI version

.. code-block:: bash

    $ sudo pip3 install --upgrade stups-senza


Command Line Usage
==================

Senza definitions can be bootstrapped conveniently to get started quickly:

.. code-block:: bash

    $ senza init myapp.yaml

Cloud Formation stacks are created from Senza definitions with the ``create`` command:

.. code-block:: bash

    $ senza create myapp.yaml 1 0.1-SNAPSHOT

You can disable the automatic Cloud Formation rollback-on-failure in order to do 'post-mortem' debugging (e.g. on an EC2 instance):

.. code-block:: bash

    $ senza create --disable-rollback myerroneous-stack.yaml 1

Stacks can be listed using the ``list`` command:

.. code-block:: bash

    $ senza list myapp.yaml       # list only active stacks for myapp
    $ senza list myapp.yaml --all # list stacks for myapp (also deleted ones)
    $ senza list                  # list all active stacks
    $ senza list --all            # list all stacks (including deleted ones)

There are a few commands to get more detailed information about stacks:

.. code-block:: bash

    $ senza resources myapp.yaml 1 # list all CF resources
    $ senza events myapp.yaml 1    # list all CF events
    $ senza instances myapp.yaml 1 # list EC2 instances and IPs
    $ senza console myapp.yaml 1   # get EC2 console output for all stack instances
    $ senza console 172.31.1.2     # get EC2 console output for single instance

Traffic can be routed via Route53 DNS to your new stack:

.. code-block:: bash

    $ senza traffic myapp.yaml      # show traffic distribution
    $ senza traffic myapp.yaml 2 50 # give version 2 50% of traffic

Stacks can be deleted when they are no longer used:

.. code-block:: bash

    $ senza delete myapp.yaml 1

Available Taupage AMIs and all other used AMIs can be listed to check whether old, outdated images are still in-use or if a new Taupage AMI is available:

.. code-block:: bash

    $ senza images


.. Tip::

    All commands and subcommands can be abbreviated, i.e. the following lines are equivalent:

    .. code-block:: bash

        $ senza list
        $ senza l

Bash Completion
---------------

The programmable completion feature in Bash permits typing a partial command, then pressing the :kbd:`[Tab]` key to autocomplete the command sequence.
If multiple completions are possible, then :kbd:`[Tab]` lists them all.

To activate bash completion for the Senza CLI, just run:

.. code-block:: bash

    $ eval "$(_SENZA_COMPLETE=source senza)"

Put the eval line into your :file:`.bashrc`:

.. code-block:: bash

    $ echo 'eval "$(_SENZA_COMPLETE=source senza)"' >> ~/.bashrc


Controlling Command Output
--------------------------

The Senza CLI supports three different output formats:

``text``
    Default ANSI-colored output for human users.
``json``
    JSON output of tables for scripting.
``tsv``
    Print tables as `tab-separated values (TSV)`_.

JSON is best for handling the output programmatically via various languages or `jq`_ (a command-line JSON processor). The text format is easy for humans to read, and "tsv" format works well with traditional Unix text processing tools, such as sed, grep, and awk:

.. code-block:: bash

    $ senza list --output json | jq .
    $ senza instances my-stack --output tsv | awk -F\\t '{ print $6 }'

.. _senza-definition:

Senza Definition
================

Senza definitions are Cloud Formation templates as YAML with added 'components' on top.
A minimal Senza definition without any Senza components would look like:

.. code-block:: yaml

    Description: "A minimal Cloud Formation stack creating a SQS queue"
    SenzaInfo:
      StackName: example
    Resources:
      MyQueue:
        Type: AWS::SQS::Queue

.. Tip::

    Use ``senza init`` to quickly bootstrap a new Senza definition YAML for most common use cases (e.g. a web application).

During evaluation of the definition, mustache templating is applied with access to the rendered definition,
including the SenzaInfo, SenzaComponents and Arguments key (containing all given arguments).

Senza Info
----------

The ``SenzaInfo`` key must always be present in the definition YAML and configures global Senza behavior.

Available properties for the ``SenzaInfo`` section are:

``StackName``
    The stack name (required).
``OperatorTopicId``
    Optional SNS topic name or ARN for Cloud Formation notifications. This can used for example to send notifications about deployments to a mailing list.
``Parameters``
    Custom Senza definition parameters. This can be used to dynamically substitute variables in the Cloud Formation template.

.. code-block:: yaml

    # basic information for generating and executing this definition
    SenzaInfo:
      StackName: hello-world
      Parameters:
        - ApplicationId:
            Description: "Application ID from kio"
        - ImageVersion:
            Description: "Docker image version of hello-world."
        - MintBucket:
            Description: "Mint bucket for your team"
        - GreetingText:
            Description: "The greeting to be displayed"
            Default: "Hello, world!"
    # a list of senza components to apply to the definition
    SenzaComponents:
      # this basic configuration is required for the other components
      - Configuration:
          Type: Senza::StupsAutoConfiguration # auto-detect network setup
      # will create a launch configuration and auto scaling group with scaling triggers
      - AppServer:
          Type: Senza::TaupageAutoScalingGroup
          InstanceType: t2.micro
          SecurityGroups:
            - app-{{Arguments.ApplicationId}}
          IamRoles:
            - app-{{Arguments.ApplicationId}}
          AssociatePublicIpAddress: false # change for standalone deployment in default VPC
          TaupageConfig:
            application_version: "{{Arguments.ImageVersion}}"
            runtime: Docker
            source: "stups/hello-world:{{Arguments.ImageVersion}}"
            mint_bucket: "{{Arguments.MintBucket}}"

.. code-block:: bash

    $ senza create example.yaml 3
    Usage: __main__.py create [OPTIONS] DEFINITION VERSION [PARAMETER]...

    Error: Missing parameter "ApplicationId"
    $ senza create example.yaml 3 example latest mint-bucket
    Generating Cloud Formation template.. OK
    Creating Cloud Formation stack hello-world-3.. OK

The parameters can also be specified by name, which might come handy in
complex scenarios with sizeable number of parameters, and also to make the
command line more easily readable, for example:

.. code-block:: bash

    $ senza create example.yaml 3 example MintBucket=<mint-bucket> ImageVersion=latest

Here, the ``ApplicationId`` is given as a positional parameter, then the two
other parameters follow specified by their names.  The named parameters on the
command line can be given in any order, but no positional parameter is allowed
to follow the named ones.

.. Note::

   The ``name=value`` named parameters are split on first ``=`` which makes it
   possible to still include a literal ``=`` in the value part.  This also
   means that if you have to include it in the parameter value, you need to
   pass this parameter with the name, to prevent ``senza`` from treating the
   part of the parameter value before the first ``=`` as the parameter name.

Any parameter may be given a default value using ``Default`` attribute.
If a parameter was not specified on the command line (either as positional or
named one), the default value is used.  It makes sense to always put all
parameters which have a default value at the bottom of the parameter
definition list, otherwise one will be forced to specify all the following
parameters using a ``name=value`` as there would be no way to map them to
proper position.

Mappings
--------

Mappings are essentially key-value pairs and behave exactly as `CloudFormation Mappings <http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/mappings-section-structure.html>`_. Use Mappings for ``Images``, ``ServerSubnets`` or ``LoadBalancerSubnets``. An Example:

.. code-block:: yaml

   Mappings:
      Images:
         eu-west-1:
            MyImage: "ami-123123"
   # (..)
   Image: MyImage

Senza Components
----------------

Components are predefined Cloud Formation snippets that are easy to configure and generate all the boilerplate JSON that is required by Cloud Formation.

All Senza components must be configured in a list below the top-level "SenzaComponents" key, the structure is as follows:

.. code-block:: yaml

    SenzaComponents:
      - ComponentName1:
          Type: ComponentType1
          SomeComponentProperty: "some value"
      - ComponentName2:
          Type: ComponentType2

.. Note::

    Please note that each list item below "SenzaComponents" is a map with only one key (the component name).
    The YAML "flow-style" syntax would be: ``SenzaComponents: [{CompName: {Type: CompType}}]``.


Senza::StupsAutoConfiguration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The **StupsAutoConfiguration** component type autodetects load balancer and server subnets by relying on STUPS' naming convention (DMZ subnets have "dmz" in their name). It also finds the latest Taupage AMI and defines an image "LatestTaupageImage" which can be used by the "TaupageAutoScalingGroup" component.

Example usage:

.. code-block:: yaml

    SenzaComponents:
      - Configuration:
          Type: Senza::StupsAutoConfiguration

This component supports the following configuration properties:

``AvailabilityZones``
    Optional list of AZ names (e.g. "eu-west-1a") to filter subnets by.
    This option is relevant for attaching EBS volumes as they are bound to availability zones.

.. _senza-taupage-auto-scaling-group:

Senza::TaupageAutoScalingGroup
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The **TaupageAutoScalingGroup** component type creates one AWS AutoScalingGroup resource with a LaunchConfiguration for the Taupage AMI.

.. code-block:: yaml

    SenzaComponents:
      - AppServer:
          Type: Senza::TaupageAutoScalingGroup
          InstanceType: t2.micro
          SecurityGroups:
            - app-myapp
          ElasticLoadBalancer: AppLoadBalancer
          TaupageConfig:
            runtime: Docker
            source: pierone.example.org/foobar/myapp:1.0
            ports:
              8080: 8080
            environment:
              FOO: bar

This component supports the following configuration properties:

``InstanceType``
    The EC2 instance type to use.
``SecurityGroups``
    List of security groups to associate the EC2 instances with. Each list item can be either an existing security group name or ID.
``IamInstanceProfile``
    ARN of the IAM instance profile to use. You can either use "IamInstanceProfile" or "IamRoles", but not both.
``IamRoles``
    List of IAM role names to use for the automatically created instance profile.
``Image``
    AMI to use, defaults to ``LatestTaupageImage``. If you want to use a different AMI, you have to create a Mapping for it.
``ElasticLoadBalancer``
    Name of the ELB resource. Specifying the ELB resource will automatically use the `"ELB" health check type for the auto scaling group`_.
``HealthCheckType``
    How the auto scaling group should perform instance health checks. Value can be either "EC2" or "ELB".
    Default is "ELB" if ``ElasticLoadBalancer`` is set and "EC2" otherwise.
``HealthCheckGracePeriod``
    The length of time in seconds after a new EC2 instance comes into service that Auto Scaling starts checking its health.
``TaupageConfig``
    Taupage AMI config, see :ref:`taupage` for details.
    At least the properties ``runtime`` ("Docker") and ``source`` (Docker image) are required.
    Usually you will want to specify ``ports`` and ``environment`` too.
``AssociatePublicIpAddress``
    Whether to associate EC2 instances with a public IP address. This boolean value (true/false) is false by default.
``BlockDeviceMappings``
    Spcify additional EBS Devices you want to attach to the nodes. See for Option Map below.
``AutoScaling``
    Map of auto scaling properties, see below.

``AutoScaling`` properties are:

``Minimum``
    Minimum number of instances to spawn.
``Maximum``
    Maximum number of instances to spawn.
``MetricType``
    Metric to do auto scaling on, only supported value is ``CPU``
``ScaleUpThreshold``
    On which value of the metric to scale up. For the "CPU" metric: a value of 70 would mean 70% CPU usage.
``ScaleDownThreshold``
    On which value of the metric to scale down. For the "CPU" metric: a value of 40 would mean 40% CPU usage.

``BlockDeviceMappings`` properties are:

``DeviceName``
    For example: /dev/xvdk
``Ebs``
    Map of EBS Options, see below.


``Ebs`` properties are:

``VolumeSize``
    How Much GB should this EBS have?

Senza::WeightedDnsElasticLoadBalancer
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The **WeightedDnsElasticLoadBalancer** component type creates one HTTPs ELB resource with Route 53 weighted domains.
The SSL certificate name used by the ELB can either be given (``SSLCertificateId``) or is autodetected.
You can specify the main domain (``MainDomain``) or the default Route53 hosted zone is used for the domain name.
By default, an internal load balancer is created. This is different from the AWS default behaviour. To create an internet-facing
ELB, explicitly set the ``Scheme`` to ``internet-facing``.

.. code-block:: yaml

    SenzaComponents:
      - AppLoadBalancer:
          Type: Senza::WeightedDnsElasticLoadBalancer
          HTTPPort: 8080
          SecurityGroups:
            - app-myapp-lb

The WeightedDnsElasticLoadBalancer component supports the following configuration properties:

``HTTPPort``
    The HTTP port used by the EC2 instances.
``HealthCheckPath``
    HTTP path to use for health check (must return 200), e.g. "/health"
``HealthCheckPort``
    Optional. Port used for the health check. Defaults to ``HTTPPort``.
``SecurityGroups``
    List of security groups to use for the ELB. The security groups must allow SSL traffic.
``MainDomain``
    Main domain to use, e.g. "myapp.example.org"
``VersionDomain``
    Version domain to use, e.g. "myapp-1.example.org"
``Scheme``
    The load balancer scheme. Either ``internal`` or ``internet-facing``. Defaults to ``internal``.



.. _AWS CloudFormation templates: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-guide.html
.. _tab-separated values (TSV): https://en.wikipedia.org/wiki/Tab-separated_values
.. _jq: https://stedolan.github.io/jq/
.. _"ELB" health check type for the auto scaling group: http://docs.aws.amazon.com/AutoScaling/latest/DeveloperGuide/healthcheck.html
