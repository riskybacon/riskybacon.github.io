---
layout: post
title: iOS SNMP Trap Push Notifications
excerpt_separator: <!--more-->
---

Push notifications are fun. If you know a bit about writing iOS apps and how to configure your app to receive push notifications, here's some code to turn SNMP traps into push notifications using <a href="http://parse.com">Parse</a>

<!--more-->

I didn't take as good of notes as I usually do, but I believe you need to install snmp-utils on the server side. If you're using CentOS 7, issue

{% highlight bash %}
yum install net-snmp-utils
{% endhighlight %}

Configure /etc/snmp/snmptrapd.conf

{% highlight bash %}
# Example configuration file for snmptrapd
#
# No traps are handled by default, you must edit this file!
#
authCommunity   log,execute,net your_supersecret_community

# the generic traps
traphandle SNMPv2-MIB::coldStart    /usr/local/bin/traphandler coldStart
traphandle SNMPv2-MIB::warmStart    /usr/local/bin/traphandler warmStart
traphandle IF-MIB::linkDown         /usr/local/bin/traphandler linkDown
traphandle IF-MIB::linkUp           /usr/local/bin/traphandler linkUp
{% endhighlight %}

Install /usr/local/bin/traphandler

{% highlight python %}
#!/usr/bin/python
# 
# Take an SNMP trap, pull it apart, send it to Parse and send out a push
# notification to iOS devices
#

import fileinput, sys, os, re
from parse_rest.connection import register
from parse_rest.installation import Push
from parse_rest.datatypes import Object

# Parse app and rest keys
parse_app_id = 'your parse app id'
parse_rest_api_key = 'your parse app rest api key'

# Register the app
register(parse_app_id, parse_rest_api_key)


# The Parse Trap object
class Trap(Object):
    pass

# Extract the source and destination IP address information
def source_and_dest(input_stream):
    source_ip = None
    source_port = None
    dest_ip = None
    dest_port = None

    # Pull the source and destination IPs out of the first two lines
    host_line = input_stream.readline()
    ip_line = input_stream.readline()

    # UDP: [10.57.0.5]:57860->[10.57.0.5]:162
    match_ipv4 = 'd{1,3}.d{1,3}.d{1,3}.d{1,3}'
    match_port = 'd{1,5}'
    regex = '[(%s)]:(%s)->[(%s)]:(%s)' % (match_ipv4, match_port, match_ipv4, match_port)
    search = re.search(regex, ip_line)

    if search:
        source_ip = search.group(1)
        source_port = search.group(2)
        dest_ip     = search.group(3)
        dest_port   = search.group(4)

    return (source_ip, source_port, dest_ip, dest_port)

# Take a line and a list of patterns to ignore. If the line
# matches any one of these patterns, return True
def should_ignore_line(line, strings_to_ignore):
    should_ignore_line = None
    for search_string in strings_to_ignore:
        if re.search(search_string, line):
            should_ignore_line = True
    return should_ignore_line

# Grab the varbind lines out of a stream
def varbind_lines(input_stream):
    # List of lines to ignore
    ignore_list = ['SNMP-COMMUNITY-MIB::snmpTrapCommunity',
                   'SNMP-COMMUNITY-MIB::snmpTrapAddress',
                   'DISMAN-EVENT-MIB::sysUpTimeInstance']

    # Pull a line out of the stream
    line = input_stream.readline()

    varbinds = []
    while line:
        # Should this line be ignored? If not, add it to the list
        if not should_ignore_line(line, ignore_list):
            varbinds.append(line)
        # Get next line
        line = input_stream.readline()

    return varbinds

# Pull the interface index (ifIndex) out of a varbind string
def find_if_index(varbind):
    if_index = None
    search = re.search('IF-MIB::ifIndex.?(d*)s*=?s*(d*)', varbind)

    if search:
        if search.group(1):
            if_index = search.group(1)
        elif search.group(2):
            if_index = search.group(2)
        
    return if_index

# Generic link up/down change handlers
def link_change(varbinds, ignore_list):
    message = ''
    if_index = None
    if_admin_status = None
    other_info = []
    for varbind in varbinds:
        if not should_ignore_line(varbind, ignore_list):
            if_index_search = find_if_index(varbind)
            if if_index_search:
                if_index = if_index_search
            else:
                other_info.append(varbind)

    if if_index:
        message = message + 'ifIndex ' + if_index + 'n'

    message = message + 'n'.join(other_info)

    return message

# Link down handler
def link_down(varbinds):
    ignore_list = ['SNMPv2-MIB::snmpTrapOID.0 IF-MIB::linkDown']
    return 'Link down: ' + link_change(varbinds, ignore_list)

# Link up handler
def link_up(varbinds):
    ignore_list = ['SNMPv2-MIB::snmpTrapOID.0 IF-MIB::linkUp']
    return 'Link up: ' + link_change(varbinds, ignore_list)

def cold_start(varbinds):
#    return 'n'.join(varbinds)
    return 'cold start'

def warm_start(varbinds):
    #return 'n'.join(varbinds)
    return 'warm start'

def auth(varbinds):
    return 'n'.join(varbinds)

def enterprise(varbinds):
    return 'n'.join(varbinds)

# Pull IP info out of incoming stream
(source_ip, source_port, dest_ip, dest_port) = source_and_dest(sys.stdin)

# Pull varbinds out of incoming strings
varbinds = varbind_lines(sys.stdin)

# OID is the first varbind
oid = varbinds[0]
message = "n".join(varbinds[1:])
message = "unhandled"

# Handle event types
event_type = sys.argv[1]

if event_type == 'linkDown':
    message = link_down(varbinds[1:])
elif event_type == 'linkUp':
    message = link_up(varbinds[1:])
elif event_type == 'warmStart':
    message = warm_start(varbinds[1:])
elif event_type == 'coldStart':
    message = cold_start(varbinds[1:])
elif event_type == 'auth':
    message = auth(varbinds[1:])
else:
    message = enterprise(varbinds[1:])

# Create Parse Trap object
trap = Trap(oid = oid, type = sys.argv[1], value = message, source_ip = source_ip, source_port = source_port, dest_ip = dest_ip, dest_port = dest_port)
# Push object out to Parse
trap.save()

# Send out push alert to iOS devices
Push.alert({"alert": source_ip + ' ' + message,
            "badge": "Increment",},
           where={'deviceType': 'ios'})
{% endhighlight %}

Make it executable and start the traphandler:
{% highlight bash %}
chmod 755 /usr/local/bin/traphandler
systemctl enable snmptrapd.service
systemctl start snmptrapd.service
{% endhighlight %}
