---
layout: post
title: FreeRADIUS Python VLAN Module
excerpt_separator: <!--more-->
---

In February 2015, I needed to assign VLANs to devices connecting to a wireless network. The server infrastructure and data was already in place. I just needed to add some glue. I decided to write a FreeRADIUS Python module to assign the VLANs.
<!--more-->
What we had:
<ul>
	<li><a href="http://www.xirrus.com/">Xirrus</a> APs using RADIUS for authentication</li>
	<li><a title="FreeRADIUS" href="http://freeradius.org">FreeRADIUS</a> providing the RADIUS service</li>
	<li>A mix of clients, mostly iPhones, Apple laptops, Linux laptops and some Windows laptops</li>
	<li><a title="OpenLDAP" href="http://www.openldap.org/OpenLDAP">OpenLDAP</a> with the FreeRADIUS schema installed, and some local schema changes. No passwords stored in LDAP</li>
	<li><a title="MIT Kerberos" href="http://web.mit.edu/kerberos">MIT Kerberos</a> for the password store</li>
	<li>The arcfour-hmac:normal hash stored in our Kerberos DB for Windows client support</li>
	<li><a title="KCRAP" href="http://http://www.spock.org/kcrap/">Kerberos Challenge Response Authentication Protocol</a> for Windows client support</li>
</ul>
Versions:
<ul>
	<li>CentOS 6.5</li>
	<li>FreeRadius 2.1.12 + kcrap patch</li>
	<li>OpenLDAP 2.4.23</li>
	<li>Kerberos 1.6.1</li>
</ul>
We needed something to:
<ul>
	<li><span style="line-height: 1.5">Query the LDAP database:</span>
<ul>
	<li><span style="line-height: 1.5">Take username and MAC address as input</span></li>
	<li><span style="line-height: 1.5">Look at LDAP groups for VLAN tags</span></li>
	<li><span style="line-height: 1.5">Output a VLAN tag</span></li>
</ul>
</li>
	<li><span style="line-height: 1.5">Insert a response into the RADIUS response packet with the VLAN tag information</span></li>
	<li>Prioritize VLAN assignment. Users can belong to multiple groups. Assign the VLAN from the group with the lowest priority number</li>
	<li>Check for known MAC addresses. The MAC address of the device must be known to assign anything other than the guest VLAN</li>
</ul>
I decided to use the FreeRADIUS module system. The documentation raises more questions than it answers. Luckily, the FreeRADIUS source code is nicely organized and readable. The section I needed to hook into is post-auth.

Steps to set up:
<ul>
	<li>Add an LDAP attribute to set a group's priority</li>
	<li>Install the Python module that takes a user ID and MAC address and returns a VLAN tag</li>
	<li>Configure FreeRADIUS to use the Python module</li>
</ul>
The last two steps are what were interesting to me. The FreeRADIUS search path isn't discussed, and the input to the module isn't described and the required output isn't documented.
<h2>FreeRADIUS Python VLAN Module</h2>
The keys to making the module work are:
<ul>
	<li>Placing the module in the right location</li>
	<li>Returning an appropriate tuple in the post_auth function</li>
	<li>Configuring FreeRADIUS to use the module</li>
</ul>
The host I was using runs CentOS 6.5 and Python 2.6. The right location for the module turned out to be /usr/lib64/python2.6/ldap2vlan.py. I'm sure there are other places that would also work.

Returning an appropriate tuple is not documented. The post-auth function needs to return a 3-tuple with the following elements:
<p class="p1"><span class="s1"><span class="Apple-converted-space">   (</span>radiusd.RLM_MODULE_UPDATED, (n-tuple of 2-tuples), ('Post-Auth-Type', 'python')</span></p>
<p class="p1">The middle tuple is where you put the key-value pairs for the RADIUS response packet. Example:</p>
<p class="p1">             ('Tunnel-Private-Group-Id', 92),</p>
<p class="p1"><span class="s1"><span class="Apple-converted-space">             </span>('Tunnel-Type', 'VLAN'),</span></p>
<p class="p1"><span class="s1"><span class="Apple-converted-space">             </span>('Tunnel-Medium-Type', 'IEEE-802'),</span></p>
<p class="p1">I decided to add some troubleshooting to the module. I did this by also making the module callable from the commandline. Examples:</p>
<p class="p1">To show the VLAN that will be assigned to a user / mac address pair:</p>

<blockquote>ldap2vlan username mac-addrees</blockquote>
Shows the VLAN-enabled groups that a user belongs to:
<blockquote>ldap2vlan username</blockquote>
Shows all groups in LDAP that provide VLAN information, sorted:
<blockquote>ldap2vlan</blockquote>
The ldap2vlan module:

[codeblock name="ldap2vlan.py"]

In <strong> /etc/raddb/modules/python</strong>:

[codeblock name = "python_module_config"]

In <strong>/etc/raddb/sites-available/default</strong>, change the post-auth section to be:

[codeblock name = "python_post_auth_hook"]

<code>Put the following into <strong>/usr/lib64/python2.6/site-packages/radiusd.py</strong>
</code>

[codeblock name = "radiusd.py"]

A better file is supposed to be generated when FreeRADIUS is built. This doesn't happen, but I found that the test file works fine.
