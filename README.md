This is a guide for how to create an SSH reverse tunnel that automatically re-establishes after disconnection.

This is useful if you have a computer without a public IP (e.g. behind NAT) and you need SSH access wihout forwarding a port from the gateway.

The client (computer behind NAT) could be a little headless single-board computer sitting at a university laboratory logging some data. You want to check in on it periodically from home. The little computer has an internet connection but you can't get a remote SSH connection from outside the university network because the public IP used by the little computer is shared and no ports are forwarded (and the IT support staff won't help you).

This solution works by having the client create an SSH connection to another computer under your control (the _server_) which _does_ have a non-shared public IP address and request a reverse tunnel. This is a feature built into OpenSSH.

# Choice of ports

The following example uses port 2222 as the forwarded port on the server and port 22 as the normal SSH port on the server.

You may encounter networks that don't allow anything other than web access. In that case port 22 won't work for you, but port 443 probably will.

To make the SSH server also listen on port 443, first ensure that your server doesn't already have an https web server using that port (it is the default web SSL/TLS port).

Then edit `/etc/ssh/sshd_config` and add the line `Port 443` after `Port 22` to make it listen on both ports.

Restart the SSH server:

```
sudo service ssh restart
```

Then in the following instructions and in the file `autossh.service` change all instances of `-p 22` to `-p 443` and _do not_ change the '22' when it appears immediately after `localhost:`.

# SSH reverse tunnel

You will need a server that is world-accessible via ssh and on which you have root access.

On the server create a user for the tunnel on the server:

```
adduser \
  --disabled-password \
  --shell /bin/false \
  --gecos "user for reverse ssh tunnel" \
  tunnel-user
```

On the server you will need to add the following to /etc/ssh/sshd_config

```
Match User tunnel-user
   AllowTcpForwarding yes
   X11Forwarding no
   PermitTunnel no
   GatewayPorts yes # Allow users to bind tunnels on non-local interfaces
   AllowAgentForwarding no
   PermitOpen localhost:2222 myserver.example.com:2222 # Change this line!
   ForceCommand echo 'This account is restricted for ssh reverse tunnel use'
```

Where you replace myserver.example.com with the public hostname or public IP of the server on which you are editing this file.

Then run:

```
sudo service ssh restart
```

On the client, log as the user that will be initiating the ssh tunnel, for our example it will be `client-user` and run:

```
ssh-keygen -t rsa
```

Hit enter to accept the default when it asks you for the key location. Now open the public key file:

```
less ~/.ssh/id_rsa.pub
```

and copy the contents into your copy-paste buffer.

On the server, create the .ssh directory and authorized keys file. Set permissions and open the file in an editor. Then paste the public key into the file and close and save.

```
cd /home/tunnel-user
mkdir .ssh
touch .ssh/authorized_keys
chmod 700 .ssh
chmod 600 .ssh/authorized_keys
chown -R tunnel-user.tunnel-user .ssh
nano .ssh/authorized_keys
# paste in the public key, hit ctrl+x then y, then enter to save
```

Now on the client:

```
ssh -p 22 -N tunnel-user@myserver.example.com
```

It will probably about the authentificy of the host, just pick "yes". If it asks for a password something went wrong. If it just sits there forever, apparently doing nothing, then everything is working as expected. Hit ctrl-c to cancel.

Now try to create a tunnel:

```
ssh -p 22 tunnel-user@myserver.example.com -N -R myserver.example.com:2222:localhost:22
```

while that is running, from e.g. your laptop try to connect to the client via the reverse tunnel:

```
ssh -p 2222 tunnel-user@myserver.example.com
```

You should get a password prompt (or a shell if you have pubkey authentication set up).

If this works you can set up autossh to make the client auto-establish the tunnel on boot and auto-re-establish this tunnel every time it fails.

Assuming you're root on the client computer, first install autossh:

```
apt install autossh
```

Then copy `autossh_loop` from this directory to `/usr/local/bin/autossh_loop` on the client computer and make it executable with:

```
chmod 755 /usr/local/bin/autossh_loop
```

Copy `autossh.service` to `/etc/systemd/system/autossh.service` and edit the file changing all occurrences of `myserver.example.com` to the hostname of your server and changing the `User=tunnel-user` line to the user that you granted access to the server. 

Save the file and run:

```
systemctl enable autossh.service
service autossh start
```

Your tunnel should now be establish and will re-establish on reboot or failure.
