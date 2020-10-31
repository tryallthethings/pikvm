# IPMI BMC
IPMI is an [old protocol](https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface) for remote server management.
It can be useful for managing a large number of machines with Pi-KVM. Its advantage is that it is supported by many enterprise systems.

:exclamation: Although Pi-KVM supports the IPMI protocol, we strongly recommend that you DO NOT USE IT
outside of trusted networks due to the protocol's [insecurity](https://github.com/NitescuLucian/nliplace.com.blog.drafts). Use Redfish or [KVMD API](api.md) instead of it.

To enable ipmi IPMI BMC follow these steps:
1. Switch the filesystem to the RW-mode:
    ```
    # rw
    ```
2. Setup IPMI account in file `/etc/kvmd/ipmipasswd`.
3. Enable `kvmd-ipmi` daemon:
    ```
    # systemctl enable --now kvmd-ipmi
    ```
4. Switch the filesystem back to the RO:
    ```
    # ro
    ```
5. Here some examples (on the remote PC):
    ```
    $ ipmitool -I lanplus -U admin -P admin -H pikvm power status
    $ ipmitool -I lanplus -U admin -P admin -H pikvm power on
    ```
    
# IPMI SoL

IPMI supports the ability to get console access to the server using Serial-over-LAN. Pi-KVM can act as a proxy for your server's COM port.

To use this feature, you will need a USB-COM adapter that you need to connect to the Pi-KVM. The COM port of the adapter need to be connected to the server. As with IPMI BMC, you need to configure `kvmd-vnc` and add the following configuration to `/etc/kvmd/override.yaml`:

```yaml
ipmi:
    sol:
        device: /dev/ttyUSB0  # Path of your USB-COM adapter
        speed: 115200
```

After enabling `kvmd-ipmi`, all requests that it receives over the network regarding the COM port will be forwarded to your server. For example:

```
$ ipmitool -I lanplus -U admin -P admin -H pikvm sol activate
```

# Redfish
[Redfish](https://www.dmtf.org/standards/redfish) is a more modern server management protocol designed to replace IPMI.
It is based on HTTP and fixes many security issues. If possible, we recommend using it instead of IPMI, or using the [KVMD API](api.md).

There're not special actions required to use Redfish. In addition, Redfish will use regular Pi-KVM credentials.
But for systems that have been upgraded to KVMD 2.0 (not a clean image installation), you will probably need to edit
the `/etc/kvmd/nginx/kvmd.ctx-server.conf` file to add these lines at the end:

```nginx
location /redfish {
      proxy_pass http://kvmd;
      include /etc/kvmd/nginx/loc-proxy.conf;
      auth_request off;
}
```

:exclamation: Don't be confused by the parameter `auth_request off`. KVMD performs authorization on its own.
The only open HTTP entrypoint is `/redfish/v1`, which returns a static document and does not change the state of the Pi-KVM. It's safe.

If there is a file in your system after the update `/etc/kvmd/nginx/kvmd.ctx-server.conf.pacnew` you can just move it:

```
# mv /etc/kvmd/nginx/kvmd.ctx-server.conf.pacnew /etc/kvmd/nginx/kvmd.ctx-server.conf
```

:exclamation: Be careful not to lose your local changes if you have done anything with this file before.

To access the Redfish API, use HTTP Basic Auth. Also you can use the [redfishtool](https://github.com/DMTF/Redfishtool):

```
$ redfishtool -S Never -r pikvm root
$ redfishtool -S Never -u admin -p admin -r pikvm Systems
$ redfishtool -S Never -u admin -p admin -r pikvm Systems reset ForceOff
```
