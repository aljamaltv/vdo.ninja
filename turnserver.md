# TURN Server Setup Guide

## Understanding STUN vs TURN

WebRTC tries to establish peer-to-peer connections using the following methods, in order:
1. Direct connection
2. STUN - helps peers discover their public IP address and port
3. TURN - relays traffic between peers when direct/STUN connections fail

### STUN (Session Traversal Utilities for NAT)
- Lightweight protocol that helps peers behind NAT discover their public IP address
- Low bandwidth usage as it only helps with connection discovery
- No relay costs, but doesn't work when peers are behind strict firewalls

### TURN (Traversal Using Relays around NAT)
- Relays all traffic through the server when direct/STUN connections fail
- Higher bandwidth usage and server costs
- Required for peers behind symmetric NATs or strict firewalls
- More reliable but should only be used as a fallback

## Quick Install

The installer script automates the complete TURN server setup process:

```bash
sudo apt-get install
sudo apt-get upgrade

# Download the installer
wget https://raw.githubusercontent.com/steveseguin/vdo.ninja/develop/turnserver_install.sh.sample
mv turnserver_install.sh.sample turnserver_install.sh
chmod +x turnserver_install.sh

# Run the installer
sudo ./turnserver_install.sh
```

The installer will:
1. Install coturn and dependencies
2. Configure system limits for high connection loads
3. Set up base TURN/STUN configuration
4. Optionally configure SSL/TLS support
5. Create systemd service for auto-start
6. Configure proper permissions

Note: You may need to configure your firewall first for Certbot to work, and I recommend running `apt update` and `apt upgrade` first.

## Basic Configuration Explained

The basic configuration (`turnserver_basic.conf`) provides a minimal but functional TURN server:

```conf
listening-port=3478        # Standard STUN/TURN port
fingerprint               # Required for WebRTC
lt-cred-mech             # Long-term credential mechanism
user=username:password    # Authentication credentials
realm=turn.example.com   # Your server's domain
server-name=turn.example.com
no-multicast-peers       # Security measure
no-stdout-log           # Disable stdout logging
```

## SSL/TLS Support (Optional)

The installer can configure SSL/TLS support which:
- Enables TURNS (TURN over TLS) on port 443
- Automatically obtains and renews SSL certificates via certbot
- Configures automatic certificate reload without server restart

## Testing Your Server

Test your TURN server at: https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/

Example configurations to test:
- STUN/TURN: `turn:your-domain:3478`
- TURNS (if SSL enabled): `turns:your-domain:443`

## Firewall Configuration

Required ports:
- 3478 TCP/UDP (STUN/TURN)
- 443 TCP/UDP (TURNS, if enabled)
- 49152:65535 TCP/UDP (Media relay ports)

### Configuring Firewall

The following can be used to configure your `ufw` firewall on Linux if needed. Adjust accordingly.

```bash
# SSH (add this first to avoid lockout)
sudo ufw allow 22/tcp    # SSH access

# Core TURN/STUN ports
sudo ufw allow 3478/tcp  # Default TURN/STUN TCP
sudo ufw allow 3478/udp  # Default TURN/STUN UDP

# If using TLS/SSL
sudo ufw allow 443/tcp   # TURN TLS
sudo ufw allow 443/udp   # TURN TLS/DTLS

# If using Certbot for SSL renewals
sudo ufw allow 80/tcp   # HTTP

# Media relay ports
sudo ufw allow 49152:65535/tcp  # TCP relay ports
sudo ufw allow 49152:65535/udp  # UDP relay ports

# Optional if you want alt-port support
sudo ufw allow 3479/tcp  # Alternative port (port+1)
sudo ufw allow 3479/udp  # Alternative port (port+1)

# Enable UFW if not already enabled
sudo ufw enable
```

## Advanced Usage

### Reloading SSL Certificates
```bash
sudo systemctl --signal=SIGUSR2 kill coturn
```

### Checking Server Status
```bash
sudo systemctl status coturn
```

### Updating Configuration
1. Edit `/etc/turnserver.conf`
2. Restart service: `sudo systemctl restart coturn`

## Common Issues

1. **Permission denied errors**
   - The installer handles this by setting proper capabilities
   - Manual fix: `sudo setcap cap_net_bind_service=+ep /usr/bin/turnserver`

2. **SSL certificate errors (701)**
   - Verify certificate permissions
   - Check certificate paths in configuration
   - Ensure certificates are readable by turnserver user

## Production Considerations

1. **System Limits**
   The installer configures higher system limits for production use:
   - File descriptors: 65535
   - System max files: 65535

2. **Monitoring**
   - Monitor bandwidth usage
   - Watch for high CPU/memory usage
   - Track active connections

3. **Security**
   - Regularly update credentials
   - Monitor for abuse
   - Keep coturn and SSL certificates up to date
   

## VDO.Ninja Configuration

You have different ways to set and specify a TURN server in VDO.Ninja; see below.

### URL Parameters

#### TURN Server Configuration
Use `&turn` to specify custom TURN servers or modify TURN behavior:

```
&turn=user;pwd;turnserveraddress
```

Examples:
- Custom TURN: `&turn=steve;setupYourOwnPlease;turn:turn.vdo.ninja:443`
- Disable TURN: `&turn=false` or `&turn=off`
- Use Twilio: `&turn=twilio`
- No STUN: `&turn=nostun`

#### Force TURN Usage
Use `&relay`, `&private`, or `&privacy` to force TURN relay usage:
- Forces the use of TURN servers instead of direct connections
- Hides IP addresses from peers
- May increase latency
- Useful for testing or privacy requirements

### Manual Configuration

#### Basic Configuration
Add to index.html before loading main.js but after webrtc.js:

```javascript
var session = WebRTC.Media;
session.version = "26.5";
session.streamID = session.generateStreamID();

session.defaultPassword = "someEncryptionKey123";
session.stunServers = [{ 
    urls: ["stun:stun.l.google.com:19302", "stun:stun.cloudflare.com:3478"]
}];

// Basic TURN Setup
session.configuration = {
    iceServers: session.stunServers,
    sdpSemantics: session.sdpSemantics
};

var turn = {
    username: "steve",
    credential: "justtesting",
    urls: ["turn:turn.obs.ninja:443"]
};
session.configuration.iceServers.push(turn);

// Force TURN relay
// session.configuration.iceTransportPolicy = "relay";
```

#### Dynamic Credentials
To use dynamic TURN credentials with PHP:

1. Rename `turn-credentials.php.sample` to `turn-credentials.php`
2. Configure the PHP file:
```php
$expiry = 86400;
$username = time() + $expiry;
$secret = '<static-auth-secret>';
$password = base64_encode(hash_hmac('sha1', $username, $secret, true));

$turn_server = "turns:<turn-server>:<https-turn-port>";
$stun_server = "stun:<stun-server>:<stun-port>";

$arr = array($username, $password, $turn_server, $stun_server);
echo json_encode($arr);
```

3. Add credential fetching code to index.html:
```javascript
try {
    session.ws = false;
    var phpcredentialsRequest = new XMLHttpRequest();
    phpcredentialsRequest.onload = function() {
        if (this.status === 200) {
            var res = JSON.parse(this.responseText);
            session.configuration = {
                sdpSemantics: session.sdpSemantics,
                iceServers: []
            };

            let phpIceServers = {
                "username": res[0],
                "credential": res[1],
                urls: []
            };
            for (let i = 2; i < res.length; i++) {
                phpIceServers['urls'].push(res[i]);
            }
            session.configuration.iceServers.push(phpIceServers);

            if (session.ws === false) {
                session.ws = null;
                session.connect();
            }
        }
    };
    phpcredentialsRequest.open('GET', './turn-credentials.php', true);
    phpcredentialsRequest.send();
} catch (e) {
    console.error(e);
    errorlog("php-credentials script Failed");
}
```

## Support

For issues or questions:
- Create an issue on the VDO.Ninja GitHub repository
- Join the VDO.Ninja Discord community

## References
- [Coturn Documentation](https://github.com/coturn/coturn/wiki/turnserver)
- [WebRTC Samples](https://webrtc.github.io/samples/)
- [VDO.Ninja GitHub](https://github.com/steveseguin/vdo.ninja)
