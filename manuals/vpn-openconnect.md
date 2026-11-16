# Connecting to `vpn.<org>` via OpenConnect & `gp-saml-gui`

## 1. Prerequisites & Installation

Install OpenConnect, the NetworkManager OpenConnect plugin:

```bash
sudo dnf install openconnect NetworkManager-openconnect NetworkManager-openconnect-gnome
```

Install `gp-saml-gui` and dependencies:

```bash
sudo dnf install python3-gobject gtk3-devel webkit2gtk3-devel
pip3 install https://github.com/dlenski/gp-saml-gui/archive/master.zip
```

See [gp-saml-gui installation](https://github.com/dlenski/gp-saml-gui#installation) for more info.

## 3. Create the NetworkManager VPN Profile

Create the VPN connection profile in NetworkManager and configure it so session cookies are held in memory only for the active session (cookie-flags=2, gwcert-flags=2):

```bash
nmcli connection add \
  type vpn \
  con-name "My-VPN" \
  ifname "*" \
  vpn-type openconnect \
  -- \
  +vpn.data "protocol=gp,gateway=vpn.<org>,reported-os=Linux,cookie-flags=2,gwcert-flags=2"
```

## 4. Create the One-Click Connection Script

Create an executable script in ~/.local/bin/vpn-connect that authenticates via gp-saml-gui, calculates the gateway TLS fingerprint, and hands the session cookie over to NetworkManager:

```bash
mkdir -p ~/.local/bin

cat << 'EOF' > ~/.local/bin/vpn-connect
#!/usr/bin/env bash
set -euo pipefail

VPN_NAME="My-VPN"
PORTAL="vpn.<org>"

notify-send -a "$VPN_NAME" "Connecting to $VPN_NAME..." "Complete Microsoft SSO login in the popup window."

# 1. Perform SAML SSO authentication and export HOST, USER, COOKIE
eval "$(gp-saml-gui --clientos=Linux "$PORTAL")"

# 2. Fetch the gateway's SHA-256 TLS public key pin
FINGERPRINT=$(echo | openssl s_client -connect "$PORTAL:443" 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform der \
  | openssl dgst -sha256 -binary \
  | openssl base64)

# 3. Update the exact portal/gateway endpoint and bring up the VPN
nmcli connection modify "$VPN_NAME" +vpn.data "gateway=${HOST#https://}"
printf 'vpn.secrets.cookie:%s\nvpn.secrets.gwcert:pin-sha256:%s\n' "$COOKIE" "$FINGERPRINT" \
  | nmcli connection up "$VPN_NAME" passwd-file /dev/stdin

notify-send -a "$VPN_NAME" "$VPN_NAME Connected" "VPN tunnel is active."
EOF

chmod +x ~/.local/bin/vpn-connect
```

## 5. Run the connection script

If everything works as expected a new VPN connection will appear on your GNOME desktop
