---
layout: post
title: Installing Puppet Agent on OS X Yosemite
---

```bash
#!/bin/sh

if [ "$(id -u)" != "0" ]; then
  echo `whoami`": this script must be run as root, please issue: \"sudo sh \" before running this script"
  exit 1
fi

# Directory to download packages to
tmp_dir=/var/tmp

# Path to puppet binary
puppet=/usr/bin/puppet

# Path to launchctl binary
launchctl=/bin/launchctl

# Path to directory services control binary
dscl=/usr/bin/dscl

# Puppet service name
puppet_service="com.puppetlabs.puppet"

# Path to the puppet "empty" dir
puppet_empty=/var/empty/.puppet

# Path to puppet plist
puppet_plist=/System/Library/LaunchDaemons/$puppet_service.plist

# Base URL for package download
base_url="http://server.with.pkg.files/puppet"

# Packages to download
packages="facter-2.4.4.pkg hiera-1.3.4.pkg puppet-3.8.1.pkg"

# Download and install packages
cd $tmp_dir
for pkg in $packages ; do
    # Download
    curl -O "$base_url/$pkg"
    # Install
    installer -pkg $tmp_dir/$pkg -target /
    # Clean up
    rm $tmp_dir/$pkg
done

# Add puppet group
$puppet resource group puppet ensure=present

# Add puppet user$puppet resource user puppet ensure=present gid=puppet shell='/sbin/nologin'

# Hide puppet user in list
# Commented out, see above
$dscl . create /Users/puppet IsHidden 1

# Create launchd service
cat << EOF > $puppet_plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>Label</key>
        <string>com.puppetlabs.puppet</string>
        <key>OnDemand</key>
        <false/>
        <key>UserName</key>
        <string>puppet</string>
        <key>GroupName</key>
        <string>puppet</string>
        <key>EnvironmentVariables</key>
        <dict>
                <key>LANG</key>
                <string>en_US.UTF-8</string>
        </dict>
        <key>ProgramArguments</key>
        <array>
                <string>/usr/bin/puppet</string>
                <string>agent</string>                <string>--no-daemonize</string>
                <string>--logdest</string>
                <string>syslog</string>
                <string>--color</string>
                <string>false</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
        <key>ServiceDescription</key>
        <string>Puppet agent service</string>
        <key>ServiceIPC</key>
        <false/>
</dict>
</plist>
EOF

$puppet resource file $puppet_empty ensure=directory owner=puppet group=puppet
$puppet resource service $puppet_service ensure=running enable=true
```
