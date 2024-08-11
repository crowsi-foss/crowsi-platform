# security considerations for reverse proxy integration

Honeypots are designed to be attacked. By attracting these attacks, we gain valuable insights and divert malicious activity away from our critical assets. However, because honeypots are intended to be compromised, they inherently carry a high-security risk. The approach behind CROWSI is to shift the honeypot application out of our asset ecosystem into dedicated backend infrastructure.

Even so, integrating a reverse proxy application into the edge-device asset is necessary. While reverse proxies are generally less prone to attacks, they can still be targeted like any software, making it important to apply security measures to further minimize risks.

This guide discusses and highlights general security considerations that should be addressed when integrating a reverse proxy application and, consequently, CROWSI into your eco-system. Where beneficial, examples will refer to a Debian system running an Nginx reverse proxy deployment.

## General Security Aspects

It is assumed that various security measures are already in place for your edge-device to protect it against the common threats every system faces. Therefore, this section will be brief, highlighting essential aspects:

1. **System Hardening**:
   - Remove unnecessary software.
   - Implement process isolation/sandboxing (e.g., for the proxy application).
   - Apply a permission concept for individual applications (limit privileges of the proxy application process).
   - Protect and monitor the file system (e.g., write protection of critical files).

2. **Logging and Monitoring**:
   - Monitor resource consumption of your proxy to prevent resource drain or detect malicious access attempts.
   - Monitor security frameworks for suspicious events.

3. **Vulnerability Management and Patching**:
   - Set-up a vulnerability monitoring and remediation framework for your system and ensure patching via a secure update channel.

Where applicable, it might be worth considering running the reverse proxy as a dedicated container to leverage the isolation and management capabilities (e.g., simple deployment/update) provided by container technology.

## Proxy Hardening

If your chosen proxy has published hardening guidelines, these should be followed closly. These guidelines should cover aspects like:

- Removal of unnecessary software modules.
- Disabling headers and error messages that could leak information to attackers.
- Protecting critical files, like the Nginx configuration file, from manipulation.

Furthermore, we recommend setting up a TCP-based reverse proxy instead of e.g. an HTTP reverse proxy to minimize processing layers on the edge device. This also allows deploying non-HTTP decoys on your CROWSI platform.

If your chosen proxy supports additional security measures, such as rate limiting or limiting simultaneous connections per IP, consider applying them as well.

## Firewalling and Bandwidth Limitations

Based on your overall system hardening, all non-essential communication should be blocked by proper firewall settings (both incoming and outgoing). However, when integrating a reverse proxy to expose CROWSI and attract attackers, the respective port needs to be open.

For example, if you're using `nftables` and running your reverse proxy on port 8080, a suitable rule configuration might look like this:

```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0;

        # Allow established and related connections
        ct state established,related accept
        
        # Accept requests on port 8080
        tcp dport 8080 accept

        # Default policy to drop all other traffic
        policy drop;
    }
    chain forward {
        type filter hook forward priority 0;
    }
    chain output {
        type filter hook output priority 0;
    }
}
```

This configuration accepts any communication on port 8080. However, you may want to consider further restrictions based on your use case, such as limiting the request count or the maximum bandwidth. This is especially relevant if your edge device is connected via mobile connectivity, where data transfer costs can be high.

For example, to limit requests to a maximum of 5 per minute, you could make the following change:
```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0;

        # Allow established and related connections
        ct state established,related accept
        
        # Rate limit connections to port 8080 (5 per minute)
        tcp dport 8080 limit rate 5/minute accept

        # Default policy to drop all other traffic
        policy drop;
    }
    chain forward {
        type filter hook forward priority 0;
    }
    chain output {
        type filter hook output priority 0;
    }
}
```

To limit the actual bandwidth, you could use a framework like `tc` or check if your reverse proxy offers relevant options. To use tc, first add markers to incoming request data within `nftables`:
```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0;
        
        # Allow established and related connections
        ct state established,related accept
        
        # Rate limit connections to port 8080 (5 per minute), mark packets for tc
        tcp dport 8080 limit rate 5/minute counter mark set 1 accept

        # Default policy to drop all other traffic
        policy drop;
    }
    
    chain forward {
        type filter hook forward priority 0;
    }
    
    chain output {
        type filter hook output priority 0;
    }
}
```

Then, configure `tc`:
```
sudo tc qdisc add dev eth0 root handle 1: htb default 30
sudo tc class add dev eth0 parent 1:1 classid 1:30 htb rate 1mbit
sudo tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip dport 8080 0xffff flowid 1:30
```
Also the use of other Linux programs like `trickle` might be worth checking out.

## Dynamic State Control of Reverse Proxy
The measures described so far should provide a solid cybersecurity foundation for your reverse proxy implementation. However, it might also be worthwhile to implement a dynamic state control mechanism for your reverse proxy.

Instead of having the reverse proxy start by default after each boot-up of your edge device, consider keeping the proxy off by default. You can then run a script after each boot-up or continuously that evaluates the desired state of your system. Based on this evaluation, the script can start or stop the reverse proxy (and configure the firewall).

Now you might be wondering why to do this. Imagine the following scenario:

You've implemented all the previously described measures and have a robust system. However, a relevant vulnerability is discovered and published by the vendor of your reverse proxy. You now need to patch this vulnerability. Integrating the patch into your overall system might take time, or you may be unable to roll out the update to your entire fleet immediately. In such cases, you might want to deactivate the reverse proxy on your edge devices until the vulnerability is resolved, and you want to do this without deploying a software update or change.

To handle scenarios like this, you might implement a state handler on your device that dynamically checks predefined conditions and, based on those, activates or deactivates the proxy.

Other scenarios where dynamic activation and deactivation of the reverse proxy might be useful include:

- No Connectivity: When there is no connection to your CROWSI deployment, there is no meaning in having the proxy running.
- Selective Exposure: You may want to expose Crowsi only on specific edge devices.

### Implementation Suggestions
The right way to implement such a state handler depends on your eco-system and the available methods of interaction with your edge-devices. However, here are some suggestions:

- Backend Service/API: Offer a backend service or API that your edge devices can query every e.g. 30 minutes to determine whether the proxy should be running. The backend can implement custom logic, such as checking the device's software version against minimum requirements or verifying against a blacklist/whitelist.

- Parameter Management: Manage the state via a device parameter that can be toggled via commands sent, for example, using the MQTT protocol.


When implementing such a state handler, consider also tracking of metrics like the inbound and outbound network consumption. If it exceeds a custom limit, the state manager can stop the reverse proxy temporarily or adjust other configurations.

Implementing dynamic state control adds another layer of security and flexibility to your reverse proxy deployment, ensuring that your system remains secure even under changing conditions.