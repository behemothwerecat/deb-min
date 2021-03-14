**2. clone your own minimal template**

`qvm-clone debian-10-minimal tpl-deb-10-min`

* If you have a HiDPI display, you might want to set the dpi early at this step to avoid having to do it over and over in derived templates:

`qvm-run --pass-io -u root tpl-deb-10-min ‘echo “Xft.dpi: 144” >> /etc/X11/Xresources/x11-common’`

**3. Set up apt-cacher-ng (optional)**

apt-cacher-ng will allow you to cache updates. This is optional but very helpful if you have many templates. Based on [Unman's notes](https://github.com/unman/notes/blob/master/apt-cacher-ng), first install the package to the cloned minimal template:

`qvm-run --pass-io -u root tpl-deb-10-min "apt-get install apt-cacher-ng"`

During install, select 'no' when prompted with "Allow HTTP tunnels through Apt-Cacher NG?" 

Mask the service in the template.  

`qvm-run --pass-io -u root tpl-deb-10-min "systemctl mask apt-cacher-ng"`

Create a qube entitled 'cache', with Template: tpl-deb-10-min, Networking: sys-whonix.

We will use [bind-dirs](https://www.qubes-os.org/doc/bind-dirs/) in the 'cache' qube. Still in dom0, make sure folder `/rw/config/qubes-bind-dirs.d` exists.

`qvm-run --pass-io -u root cache "mkdir -p /rw/config/qubes-bind-dirs.d"`

Create a file `/rw/config/qubes-bind-dirs.d/50_user.conf` with root rights.

`qvm-run --pass-io -u root cache "touch /rw/config/qubes-bind-dirs.d/50_user.conf"`

Edit the file 50_user.conf to append a folder or file name to the binds variable. In this case we need three folders to be persistent: 

`qvm-run --pass-io -u root cache "printf \"binds+=( '/var/cache/apt-cacher-ng' )\nbinds+=( '/var/log/apt-cacher-ng' )\nbinds+=( '/etc/apt-cacher-ng' )\n\" > /rw/config/qubes-bind-dirs.d/50_user.conf"`

Reboot the cache qube. 

Use /rw/config/rc.local to start the apt-cacher-ng service:

`qvm-run --pass-io -u root cache "printf \"systemctl unmask apt-cacher-ng\nsystemctl start apt-cacher-ng\n/sbin/iptables -I INPUT -p tcp --dport 8082 -j ACCEPT\n\" >> /rw/config/rc.local"`

Edit the port number in /etc/apt-cacher-ng/acng.conf:

`qvm-run --pass-io -u root cache "sed -i -- 's/# Port:3142/Port:8082/g' /etc/apt-cacher-ng/acng.conf"`

Restart service.

`qvm-run --pass-io -u root cache "systemctl restart apt-cacher-ng"`

*I get an error here. When I check status:*

`qvm-run --pass-io -u root cache "systemctl status apt-cacher-ng.service"`
```
___ apt-cacher-ng.service - Apt-Cacher NG software download proxy
   Loaded: loaded (/lib/systemd/system/apt-cacher-ng.service; disabled; vendor preset: enabled)
   Active: failed (Result: exit-code) since Sun 2021-03-14 15:51:41 EDT; 10s ago
  Process: 724 ExecStart=/usr/sbin/apt-cacher-ng -c /etc/apt-cacher-ng ForeGround=1 (code=exited, status=217/USER)
 Main PID: 724 (code=exited, status=217/USER)

Mar 14 15:51:41 cache systemd[1]: apt-cacher-ng.service: Service RestartSec=100ms expired, scheduling restart.
Mar 14 15:51:41 cache systemd[1]: apt-cacher-ng.service: Scheduled restart job, restart counter is at 5.
Mar 14 15:51:41 cache systemd[1]: Stopped Apt-Cacher NG software download proxy.
Mar 14 15:51:41 cache systemd[1]: apt-cacher-ng.service: Start request repeated too quickly.
Mar 14 15:51:41 cache systemd[1]: apt-cacher-ng.service: Failed with result 'exit-code'.
Mar 14 15:51:41 cache systemd[1]: Failed to start Apt-Cacher NG software download proxy.
```



Set this as updateProxy in /etc/qubes-rpc/policy/qubes.UpdatesProxy

Debian templates will use this for updates with no further configuration.

For [TLS support](https://askubuntu.com/questions/1307210/how-to-get-apt-cacher-ng-to-download-and-cache-packages-from-apt-https-repositor), we will change the repository definition FROM:
`https://yum.qubes-os.org/` 
TO: 
`http://HTTPS///yum.qubes-os.org/`

Without any other changes to the apt-cacher configuration the qube will
use HTTP to the proxy which will use TLS to pick up the packages and
cache any responses. In dom0:

`qvm-run --pass-io -u root tpl-deb-10-min "sed -i -- 's#https://#http://HTTPS///#g' /etc/apt/sources.list"`

`qvm-run --pass-io -u root tpl-deb-10-min "sed -i -- 's#https://#http://HTTPS///#g' /etc/apt/sources.list.d/*.list"`

**4. Set up your own minimal template**

Run all updates:

`qvm-run --pass-io -u root tpl-deb-10-min “apt update && apt full-upgrade -y”`

...
