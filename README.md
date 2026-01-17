# How to Create a PfSense OpenVPN Server (Step-by-Step, with “Any Interface” Fix)

This guide explains, from scratch, how to set up a fully working OpenVPN Road Warrior (remote access) server on pfSense, **avoiding common “invisible” issues that can break connectivity** (like the “bind to WAN” vs “Any” interface problem).

---

## What You’ll Need

- Admin access to pfSense WebUI
- Working WAN connection (pfSense WAN has public IP)
- Your LAN subnet (e.g. 192.168.1.0/24)
- Optionally, Dynamic DNS configured for easier remote access

---

## Step 1: Install OpenVPN Package (if needed)

On most pfSense installs, OpenVPN is built-in.  
**No separate package needed unless you want additional plugins.**

---

## Step 2: Create Certificate Authority & Certificates

### 2.1 Create a Certificate Authority (CA)

1. Go to **System > Cert. Manager > CAs**
2. Click `Add`
3. Set:
    - **Descriptive Name**: OpenVPN_CA
    - **Method:** Create an internal Certificate Authority
    - Fill in your country, common name, etc. (use server’s hostname or “OpenVPN-CA” as Common Name)
4. Click `Save`

### 2.2 Create Server Certificate

1. Go to **System > Cert. Manager > Certificates**
2. Click `Add`
3. Set:
    - **Descriptive Name:** OpenVPN_Server
    - **Method:** Create an internal Certificate
    - **Certificate Authority:** Select `OpenVPN_CA`
    - **Type:** Server Certificate
    - **Common Name:** OpenVPN_Server or your pfSense hostname
4. Click `Save`

### 2.3 Create User Certificates (for VPN users)

_For each user:_
1. Go to **System > User Manager > Users**
2. Click `Add`
3. Fill out username/password, e.g., `johndoe`
4. Check “Click to create a user certificate…”
5. **Certificate authority:** `OpenVPN_CA`
6. **Type:** `User Certificate`
7. Click `Save`

---

## Step 3: Configure the OpenVPN Server Wizard

1. Go to **VPN > OpenVPN > Wizards**
2. **Type of Server:** Local User Access
3. **Certificate Authority:** `OpenVPN_CA`
4. **Server Certificate:** `OpenVPN_Server`
5. **Interface:**  
   - **Critical:** Select `Any`  
6. **Protocol:** `UDP4` (UDP on IPv4 only)
7. **Local Port:** `1194` (default; you can change if desired)
8. **Description:** (e.g., "Road Warrior VPN Server")

9. **Tunnel Network:**  
    - Enter a network not used elsewhere, e.g., `10.8.0.0/24`
    
10. **Local Network:**  
    - Enter your LAN network, e.g., `192.168.1.0/24`
    
11. **Concurrent Connections:** (As needed)
12. Tick "Inter-Client Communication" if you want VPN clients to talk to each other.
13. **Redirect Gateway:** (Recommended. Forces all client traffic through VPN)  
14. DNS Servers: Enter your LAN DNS server or pfSense’s IP (e.g., 192.168.1.1)

15. **Cryptography Settings:**  
    - Defaults are typical, but use at least AES-256-CBC or AES-256-GCM for cipher.

16. **Client Settings:**  
    - NetBIOS, DNS, etc., as needed.

### Wizard: Automatic Firewall/NAT Rules  
- Accept the Wizard's prompts to create firewall and NAT rules.

17. **Finish Wizard**
    - Click through to finish and apply settings.

---

## Step 4: Confirm/Correct Port Forward & Outbound NAT

- Go to **Firewall > NAT > Port Forward**  
  - Confirm a rule exists:  
    - WAN, UDP, Port 1194,  
    - Destination: pfSense WAN address,
    - Redirect target IP: pfSense LAN IP (e.g., 192.168.1.1),
    - Target Port: 1194
- Go to **Firewall > Rules > WAN**
    - Confirm a firewall rule exists to allow UDP 1194 to "WAN address"
- Go to **Firewall > NAT > Outbound**
    - Use “Automatic” or “Hybrid Outbound NAT”.
    - No custom rule needed unless you have advanced config.

---

## Step 5: Restart/Apply and Test

1. Go to **Status > Services**  
    - Restart the OpenVPN server service after all config changes.
2. Export client configuration:
    - Go to **VPN > OpenVPN > Client Export Utility** (install if not present via `System > Package Manager > Available Packages`)
    - Download **user-locked** or **inline** config for each user.
3. Test from external network (not LAN).

---

## Step 6: Troubleshooting

- If you time out and see no OpenVPN log entries but packet capture **does** show UDP/1194 traffic…  
    - **Edit your OpenVPN server: make sure interface is set to “Any”**
    - Restart OpenVPN service.
- Common symptoms and fixes included at the end of this file.

---
## Step 7: Add Users (later)

Repeat Step 2.3 for each new user.
- Download/export their client config (incl. cert) from the Client Export utility.

---

## 🔑 Key Settings Reference

- OpenVPN Server:
    - **Interface:** Any
    - **Protocol:** UDP
    - **Port:** 1194 (or custom)
    - **Tunnel Network:** 10.8.0.0/24
    - **Local Network:** 192.168.1.0/24
    - **Server Certificate/CA:** as created above

---

## Troubleshooting Table

| Problem                                   | What to Check/Do                                                      |
|--------------------------------------------|-----------------------------------------------------------------------|
| Port scan shows 1194 open, but won’t connect | Confirm OpenVPN binds to “Any”; see packet capture & logs             |
| Can connect from LAN only                  | Check NAT/firewall/port forward on pfSense; test from real WAN        |
| OpenVPN log shows no attempts              | Bind to “Any”, restart OpenVPN, confirm port forward/firewall rule    |
| Logs show “AUTH” or handshake, but no traffic after | Check user certificate, user password, CA trust, or client config    |
| LAN access but no internet via VPN         | Set “Redirect Gateway” & configure DNS in OpenVPN server settings     |

---

## Resources

- [Netgate: OpenVPN Road Warrior Guide](https://docs.netgate.com/pfsense/en/latest/vpn/openvpn/road-warrior.html)
- [pfSense OpenVPN Forum](https://forum.netgate.com/)

---

**Save this for your next reinstall or upgrade!**  
If you get stuck, always try packet capture, review the OpenVPN server log, and make sure you’re binding to the right interface.

```
