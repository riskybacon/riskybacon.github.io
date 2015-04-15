---
title: Synthesizing SNMP Traps
layout: post
---

While testing out some SNMP trap handling code, I became tired of going into the server room, unplugging a cable from the switch and watching the logs for trap entries and seeing how our custom SNMP trap handler worked. I'd rather be sitting in the comfort of my office, balcony, coffee shop, or anywhere else than some cold server room!

Here's a syslog entry for a link down SNMP trap from a Force10 switch:

```bash
Apr 3 09:11:29 localhost snmptrapd[29065]: 2015-04-03 09:11:29 [UDP: [10.0.5.29]:162->[10.0.5.27]:162]:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (913904900) 105 days,
  18:37:29.00
SNMPv2-MIB::snmpTrapOID.0 = OID: IF-MIB::linkDown
IF-MIB::ifIndex.36487226 = INTEGER: 36487226
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.2 = STRING: "OSTATE_DN: Changed interface state to down: Gi 0/10"
```

The goal is to synthesize this with the snmptrap command.

Sending a link down trap:

```bash
snmptrap -v 2c -c secretcommunity '' '' \
   IF-MIB::linkDown \
   ifIndex.36487226 i 36487226
```

The resulting syslog entry:

```bash
Apr 3 10:40:08 ns-host snmptrapd[29065]: 2015-04-03 10:40:08 localhost [UDP: [127.0.0.1]:55232->;[127.0.0.1]:162]:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (85644481) 9 days, 21:54:04.81
SNMPv2-MIB::snmpTrapOID.0 = OID: IF-MIB::linkDown
IF-MIB::ifIndex.36487226 = INTEGER: 36487226
```

Close, but doesn't quite look like a Force10 trap. The descriptive text portion is missing. Add it with:

```bash
snmptrap -v 2c -c secretcommunity '' '' \
  IF-MIB::linkDown \
  ifIndex.36487226 i 36487226 \
  SNMPv2-SMI::enterprises.6027.3.1.1.4.1.2 s \
    "OSTATE_DN: Changed interface state to down: Gi 0/10"
```

Which results in a log entry that looks a lot like a trap that was created by a Force10 switch:

```bash
Apr 3 10:41:19 ns-host snmptrapd[29065]: 2015-04-03 10:41:19 localhost [UDP: [127.0.0.1]:59788-&gt;[127.0.0.1]:162]:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (85651574) 9 days, 21:55:15.74 SNMPv2-MIB::snmpTrapOID.0 = OID: IF-MIB::linkDown
IF-MIB::ifIndex.36487226 = INTEGER: 36487226
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.2 = STRING: "OSTATE_DN: Changed interface state to down: Gi 0/10"
```

How about synthesizing a more complex, Force10 enterprise specific trap?

```bash
snmptrap -v 2c -c 0mgp0n13s '' '' \
SNMPv2-SMI::enterprises.6027.3.1.1.4.0.11 \
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.1 i 6 \
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.2 s "RPM_STATE: RPM1 is in Standby State." \
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.3 i 1 \
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.4 i -1
```

And the resulting log entry:

```bash
Apr  3 11:13:36 ns-host snmptrapd[29065]: 2015-04-03 11:13:36  [UDP: [10.0.5.29]:162-&gt;[10.0.5.27]:162]:
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (914637588) 105 days, 20:39:35.88
SNMPv2-MIB::snmpTrapOID.0 = OID: SNMPv2-SMI::enterprises.6027.3.1.1.4.0.11
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.1 = INTEGER: 6
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.2 = STRING: "RPM_STATE: RPM1 is in Standby State."
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.3 = INTEGER: 1
SNMPv2-SMI::enterprises.6027.3.1.1.4.1.4 = INTEGER: -1
```

>Now I can get back to writing a custom Python SNMP trap handler without sitting next to the switch gear.

## References

* Net-SNMP Tutorial -- traps [http://www.net-snmp.org/tutorial/tutorial-5/commands/snmptrap.html](http://www.net-snmp.org/tutorial/tutorial-5/commands/snmptrap.html)
* net-snmp snmptrap tutorial [http://www.net-snmp.org/wiki/index.php/TUT:snmptrap](http://www.net-snmp.org/wiki/index.php/TUT:snmptrap)
