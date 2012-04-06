Module: shared_ca
=================

Module to aid in the creation of a shared CA Puppet Infrastructure


Goal or Use Case
----------------
Have a central CA/Console host on your network that any number of masters can submit inventory to (over REST). These masters should be able to sign their own agents, so agents don't need connectivity to the console host, amongst other potential reasons.

Also, each master's ActiveMQ server should particpate in a shared broker mesh, including the console host, so orchestration can be done throughout the environment, including console live management.


Concepts
--------
In order to satisfy these goals, the CA living on the console host will be replicated out to each Puppet Master wishing to participate in the shared environment. Each of those masters CA (created by the PE installer script) will be replaced with the CA of the console host. The masters certficates (again, created at install) also need destroyed and re-signed by the shared CA.

ActiveMQ and MCollective will participate in a similar fashion but Puppet and the pe_mcollective module will handle certificate management here. Because we want the broker mesh to be automatically managed by Puppet, we'll be using a slightly modified pe_mcollective module than what ships with PE. It includes code to manage the brokers, functionality intended for a future release.

The shared_ca module aids in some of these tasks, mostly file copies & deletions.


Pre-Requisites
--------------

1. Install PE 2.5 onto a host with the master, agent & console roles selected (you'll get CA for free). Referred to as the Console host.

2. Install PE 2.5 on any number of hosts with just the master & agent roles (no console). Referred to as Master hosts.

3. Populate some content for the shared_ca module.
  Copy the following data from the console host into this modules files directory:
a)  /etc/puppetlabs/puppet/ssl/ca -- CA Directory
b)  /etc/puppetlabs/mcollective/credentials -- MCollective Credentials

You'll also need to copy in the pe_mcollective module we provided you until a compatible version makes it way into a PE 2.5.x release.
c)  pe_mcollective -- Module that handles ActiveMQ & MCollective


Workflow -- add bit about inventory service, auth.conf
------------------------------------------------------

Preparing the Shared CA
-----------------------
Once those prerequisites are met, you should run puppet apply against this module on each of your systems to prepare the hosts. For example, if you place this module into /root on your target systems during the bootstrap process:

1. puppet apply --modulepath=/root:/opt/puppet/share/puppet/modules --certname=your_machines_certname --execute 'include shared_ca'

--modulepath needs to include the directory where the stdlib module lives. The example includes the folder stdlib lives in PE 2.5.0. --certname is required if your installed certificate name differs from your hostname.

--execute 'include shared_ca' declares the modules only class which takes some actions based on facts found in /etc/puppetlabs/facter/facts.d/

On a Console host, this declared class will:
  * Stop services: pe-puppet, pe-httpd, pe-mcollective & pe-activemq
  * Copy the pe_mcollective module to /etc/puppetlabs/puppet/modules
  * Purge MCollective Certificates
  * Purge an old create_resources function that shipped with PE 2.5 by accident.

On a Master host, this declared class will:
  * Do the same as on a Console host, plus:
  * Purge the hosts entire $cadir & replace it with your Console hosts $cadir
  * Replace /etc/puppetlabs/mcollective/credentials with one from your Console host
  * Purge the Master host certificate/key pair that were created during install.


Bringing up the Modified Host
-----------------------------
Once the module has done it's business, you have two paths to continue.


A. On a Console Host:
  1. Start the pe-httpd service.
  2. Run 'puppet agent -t' which should now regenerate MCollective certificates and restart pe-mcollective and pe-activemq.
  2a. If you're not autosigning certificates, you will need to sign the MCollective certificate. Exec[check_for_signed_broker_cert] will fail on your first Puppet run, indicating that the certificate has not been signed.
  2b. puppet cert --list & puppet cert --sign $certname.pe-internal-broker
  2c. Run 'puppet agent -t' again to finish the process.
  3. Optionally turn back on the pe-puppet service.


B. On a Master Host:
  1. Start a Puppet Master manually, as it needs to generate and sign a new cert from your shared CA.
  1a. puppet master --no-daemonize --debug
  1b. Once that's done (you should see it startup successfully), you can control+C the process.
  2. Start the pe-httpd service.
  3. Run 'puppet agent -t' which should now regenerate MCollective certificates and restart pe-mcollective and pe-activemq.
  3a. If you're not autosigning certificates, you will need to sign the MCollective certificate. Exec[check_for_signed_broker_cert] will fail on your first Puppet run, indicating that the certificate has not been signed.
  3b. puppet cert --list & puppet cert --sign $certname.pe-internal-broker
  3c. Run 'puppet agent -t' again to finish the process.
ctive certificate. puppet cert --list & puppet cert --sign as appropriate.
  4. Optionally restart the pe-puppet service.

TO-DO: Document ActiveMQ scaling. Most content is already in the pe_mcollective module, really just need the activemq_brokers=broker1,broker2 variable behavior.