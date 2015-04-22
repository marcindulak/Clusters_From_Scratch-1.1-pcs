-----------
Description
-----------

This is Vagrantfile performing the configuration described in
http://clusterlabs.org/doc/en-US/Pacemaker/1.1-pcs/html/Clusters_from_Scratch
version from Thu Mar 5 16:46:11 AEDT 2015.


------------
Sample Usage
------------

Assuming you have Vagrant installed from https://www.vagrantup.com/downloads.html::

	$ git clone https://github.com/marcindulak/Clusters_From_Scratch-1.1-pcs.git
        $ cd Clusters_From_Scratch-1.1-pcs
	$ vagrant up

When done, destroy the test machines with::

	$ vagrant destroy -f


------------
Dependencies
------------

None


-------
License
-------

BSD 2-clause


----
Todo
----

1. figure out how to use fence-virt (chapter 8)
2. the configuration breaks at section 9.2
