---
layout: page
title: Driver Deep Dive
category: tut
order:  6
---

In this section we'll take a look at some of the design principles mechanics of
Shell drivers. To get started, we should first create a Shell project to experiment with.
Once again we'll use ShellFoundry.

{% highlight bash %}
shellfoundry new driver-example --template=resource-shell-clean
cd driver_example
{% endhighlight %}

#### The driver structure

The **_src_** folder in the Shell project is the driver source code root. On some IDEs
you need to set that manually when opening the Shell project.

Every Shell driver is required to have one main driver class. This class represents
the main driver interface and is the file CloudShell introspects to get and validate
the list of commands for the Shell. Every ShellFoundry template will generate the
driver main file in the **_driver.py_** file and a main driver class with the name
**_[ShellName]Driver_**, both of which are the convention.

Any public function added to this driver will be considered a Shell command. Any private function (prefixed by Python convention
by an underscore) added to the main class will be considered an internal helper function and will be ignored by CloudShell. We can
refer to these public functions in the driver class as CloudShell command functions. Each command function can also have a special
parameter called _context_. This parameter will be provided by CloudShell as an object containing information about the sandbox
and resource/app the device is executing against. Another special property of the _context_ parameter is that its completely
invisible to users or API clients executing the command. If you execute the command from the UI or API you will never see a
_context_ parameter. The _context_ object is reviewed in the
 [Getting information from CloudShell](/devguide/tut/getting-information-from-cloudshell.html) section of this guide.

The **_drivermetadata.xml_** file, also located in the _src_ directory provides additional
information on the driver functions and how CloudShell should expose and display them as CloudShell
Commands. More information about this file and how to customize CloudShell commands can be found in the
[Commands visibility and usability](/devguide/tut/customizing-driver-commands.html) section.

The driver can contain additional files and folders which you can reference in your driver class.
In order to reference a file in a nested folder you need to add an  **_\_\_init\_\_.py_** file according
to Python convention so that the folder is considered a package with its own namespace.

#### Referencing other packages

A driver can also have a **_requirements.txt_** file to declare any dependencies on external packages
just like any Python package. When CloudShell creates a virtual environment to run the driver,
it will use **_pip_** to pull those requirements and prepare the environment.

The requirements file follows the normal pip rules
 and [file format](https://pip.pypa.io/en/stable/reference/pip_install/#requirements-file-format).

As the requirements are pulled each time a driver virtual environment is created, it is recommended to either
pin the external package version in the requirement file, or at least use semantic versioning
convention to limit the automatic updates to patch releases only. If you link to an external package by name only
and without any version constraints and a new version of the package with breaking changes is released, the
driver code may break as well. All Quali released drivers either pin the dependency version or specify
a range allowing patch version updated only. For example, if you look at the generated requirements file for
our project:

{% highlight python %}
cloudshell-shell-core>=2.0.0,<2.1.0
{% endhighlight %}

According to this constraint, pip will not automatically install any minor version beyond 2.0 but patch versions are allowed.

#### Returning Results

Any result returned by the Python driver command function will be considered the command result by CloudShell.
String will be returned as is, if you return an object CloudShell will try to convert it into a string JSON.
Its not advisable to return complex objects that cannot be serialized such as sessions or connections as that would most
likely result in an error.

To demonstrate this, lets add a couple of functions to the driver:

{% highlight python %}
{% github_sample /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py 22 35 %}
{% endhighlight %}
{% github_sample_ref /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py %}

In the first example, we just return a line of string. In the second example, we actually return the context object that is
being passed to the driver.

After adding the functions reinstall the Shell using ShellFoundry and run the new commands:

![Simple String Result]({{ site.url }}/devguide/assets/simple_string_return.png)

![Simple String Result]({{ site.url }}/devguide/assets/complex_object_result.png)

Returning a result has several effects:

* The result will appear in the environment output and be linked from the commands panel (unless the command was executed with
    the no-output parameter)

* If another command or an orchestration script called this command, it will receive back the result.

* The command will appear as having completed successfully in the commands panel:


#### Handling failure

If the command failed you can throw an exception as you would in any Python code. The exception will be parsed
and the message property of the exception will be shown to the user.

![Command Results]({{ site.url }}/devguide/assets/failed_command.png)

{% highlight python %}
{% github_sample /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py 35 41 %}
{% endhighlight %}
{% github_sample_ref /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py %}

If the command is being called from an orchestration script or a different Python driver, calling a command that fails in
this way will result in an exception of type CloudShellAPIError. The error message will include the type of exception (the original
    exception raised in the code) as well as the error message.

![Command Results]({{ site.url }}/devguide/assets/failed_api_error.png)

#### Cancellation and termination

In CloudShell, cancellation and termination are treated very differently.

* **Termination** is a brute force action that will terminate the driver execution. It is intended to deal with hanging or stuck
drivers. The danger with termination is that if used lightly it may cause the driver process to be killed in the middle of a critical
process, leaving the device/driver in an unexpected or unsupported state. Because termination is a last resort unsafe action,
CloudShell restricts the permissions for this action and only domain and system admins can terminate a command. Because it can have unwanted
side effects, the termination button has an exclamation mark next to it and you'll need to confirm the action.

* **Cancellation** is only available for driver commands implementing this capability. Cancelling a command signals it to complete
and gives it a chance to abort or rollback gracefully. Drivers should implement cancellation for commands that can
take up a long period of time.

To implement cancellation for a driver command, you first need to add a parameter to the Python function, which like the
_context_ object will be passed in automatically by CloudShell. Adding the extra parameter, called _cancellation_context_
will signal to CloudShell that the command supports cancellation and the user will be able to attempt to stop it
while it is running. Next, add an entry for the command in the _drivermetadata.xml_ file and add the attribute:
**_EnableCancellation="true"_**

{% highlight xml %}
{% github_sample /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/drivermetadata.xml 0 10 %}
{% endhighlight %}
{% github_sample_ref /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/drivermetadata.xml %}

Copy the following function to the driver to add a cancellable command:

{% highlight python %}
{% github_sample /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py 43 55 %}
{% endhighlight %}
{% github_sample_ref /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py %}

Re-install the shell. You'll now see the commands pane has a regular stop button (without an exclamation mark). If you click the
stop button the _is_cancelled_ property on the _cancellation_context_ object will be updated and the driver will get a chance to
complete its current actions and end its execution.

#### Drivers and concurrency

CloudShell supports two modes for drivers:

* A serial mode of execution in which for each app/resource driver instance CloudShell will send one command at a time.
    With this mode, you can be sure as you're processing a command that no command will run in parallel with that
    same resource/app and using that driver instance object.

* Concurrent execution (default) - with this mode enabled, it will be up to the driver to handle any synchronization between
    parallel threads if needed. CloudShell will send commands to the driver in parallel.

The driver concurrency mode is defined in the shell datamodel. Open the _shell_model.xml_ file in the _datamodel_ directory.
Notice the _\<ResourceModel>_ element below. The _SupportsConcurrentCommands_ will set the concurrency mode for the Shell
driver.

{% highlight xml %}

<Shell>
    <ShellModel Family="Generic Resource">
        <ResourceModel Name="DriverDeepDive" Description="" SupportsConcurrentCommands="true">
{% endhighlight %}

For simple drivers, which have relatively short operations it makes sense to set concurrency to false and not have to
worry about operations happening in parallel. This is especially true if the Shell resource is used exclusively within each sandbox
so its not anticipated that several users will try to use the same resource in parallel.


#### Using the CloudShell API

In general, it is recommended to use the CloudShell API as little as possible from the driver, which is supposed to mostly
communicate with the app or device. However, there are several common use cases where drivers would use the API to update
CloudShell regarding the status of the resource or write additional messages to the console.

Starting a CloudShell API session requires information regarding the sandbox reservation and the server connectivity information.
All of that information is available in the _context_ parameter of the command function that is being passed by CloudShell.
For more information about that object and the information it provides please see the
[Getting information from CloudShell](/devguide/tut/getting-information-from-cloudshell.html) section.

First, as we'll need the CloudShell API, we should add it to the _requirements.txt_ file. Open the _requirements.txt_ file
and add the following line:

cloudshell-automation-api>=7.0.0.0,<7.0.1.0

To start a CloudShell API session, first import the _CloudShellAPISession_ module, then initialize an object with the
connectivity parameters. Import the _CloudShellAPISession_ module by adding the following statement to the beginning of the
_driver.py_ file:

from cloudshell.api.cloudshell_api import CloudShellAPISession

To log in we don't need a username/password, since we get a token we can use with the context object.
Copy and paste the following function:

{% highlight python %}
{% github_sample /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py 56 64 %}
{% endhighlight %}
{% github_sample_ref /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py %}

This is pretty strait forward but can also probably get repetitive. That's why its encouraged that you use some of the helper
classes that are provided with the _cloudshell_shell_core_ package. The _CloudShellSessionContext_ allows easily creating
a session from a context object. To user the helper first import the module by adding this to the _driver.py_ imports:

from cloudshell.shell.core.cloudshell_session import CloudShellSessionContext

Then, paste the following function:
{% highlight python %}
{% github_sample /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py 66 76 %}
{% endhighlight %}
{% github_sample_ref /QualiSystems/devguide_examples/blob/driver_deep_dive/adding_examples/driver_deep_dive/src/driver.py %}

Now that we have a CloudShell API session, there are three main things we may want to do with it from our driver:
Decrypt a password attribute, update the resource live status or update the console widget with progress report.
You can find the code for these operations in the examples section of the guide.