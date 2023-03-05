I had a problem that my laptop (Dell Latitude 7280) with Gentoo system, could
connect to some wifi networks, but the connection was basically not working.
It was the case for 2.4GHz networks. I found the solution to this problem
on Gentoo wiki [here](https://wiki.gentoo.org/wiki/Iwlwifi#No_internet_connection).

I basically edited: `/etc/modprobe.d/iwlwifi.conf`:

```
options iwlwifi 11n_disable=1
```
