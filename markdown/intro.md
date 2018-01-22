## Extending C++ with Python

A Brief Odyssey

* * *

Tim Serong | [tserong@suse.com](mailto:tserong@suse.com)

linux.conf.au 2018

Note: The journey is often more interesting than the destination, so I'm going
to try to take you through one of my little coding journeys.  There will be code
snippets.  I'm going to talk first about why you might want to do this, then
explain some general techniques, then show how those techniques were applied
in a specific project -- Ceph -- then cover the remaining weirdness of my
personal odyseey hacking on this code.


# Ceph

Note: For this to make any sense, I have to talk a little bit about Ceph first,
but I'm not going to show the classic block diagrams, because everyone's already
seen those, and they're not that relevant to the techniques presented here
anyway.  Instead I'll just say it's a massively scalable distributed storage
system, which you run on many node, with many disks in each.  It's awesome.
You can hear more about it in Sage's talk on Friday after lunch.


# C++

Note: The core of Ceph consists of several daemons written in C++, and some command line tools written in Python.  In order to "do stuff", the python command line tools send and receive chunks of JSON to and from the C++ daemons over a local socket.  Originally, on the daemon side, we had the MONs (which maintain the cluster map, and some other stuff) and the OSDs which manage individual disks on which data is stored, plus a couple of others.  More recently (first in Kraken, and now required in Luminous), a new daemon was added called ceph-mgr.  There were two reasons for this -- one was to offload processing of some stuff that was interesting to users from the MONs to another daemon to simplify the MON implementation and restrict it to things critical for the functioning of the cluster only.  The other reason was to add the ability to extend core ceph functionality by writing python code, rather than having to write C++.


# Python

Note: There's various advantages to this; ease of development, available libraries, efficiency of embedding python rather than marshalling JSON, some other stuff.  It means that we can easily add new functionality to the core of ceph, either new command line commands, or new long-running (daemon type) functionality in a context in which that code can see the entire cluster state.  This means we could embed a REST API inside Ceph, for example, so that other third party tools could talk to it to configure the cluster.  Or a tool to look at the data usage in the cluster and rebablance things etc.  And all this could be done in neat little modules, with some callbacks to get cluster state, and some notifications of when things change, and the option to run as a server, or just to implement some additional one-shot commands.  So that's why you might want to extend C++ with python; you have a large complicated C++ application, and you want to be able to easily add new modular functionality to it, that doesn't necessarily need all the C++ magic, or wants to take advantage of other neat python libraries.

