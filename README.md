# Cloudflare Dynamic DNS Update Script for ASUS RT-AC68U

The ASUS RT-AC68U router with the Asuswrt-Merlin custom firmware adds support for custom dynamic DNS providers. This is great for Cloudflare users because, although Cloudflare is not one of the built-in providers, we can add support for it. This script does exactly that.

Features include:
  - Support for querying your Cloudflare DNS zone to determine record IDs
  - Handling of Merlin firmware success/failure callbacks
  - Logging of JSON responses for later inspection, and
  - Configurable rate-limiting to comply with Cloudflare API TOS

## How to Configure

### Prerequisites
You should have your Merlin-enabled ASUS RT-AC68U router configured for your network with Internet access.

### Configuration Overview
Configuration of Cloudflare DDNS involves changes made on the router web portal as well as changes made on the router shell.
1. [Enable shell access and JFFS partition](#enable-shell-access-and-jffs-partition)
2. [Install Cloudflare DDNS script](#install-cloudflare-ddns-script)
3. [Enable custom DDNS](#enable-custom-ddns)
4. [Verification](#verification)
5. [Clean up](#clean-up)

##### Enable Shell Access and JFFS Partition
In the router portal, under Administration -> System,
- Basic Config -> Enable JFFS custom scripts and configs: Yes
- Service -> Enable SSH: LAN only

Save the configuration. Ensure you are able to SSH into your router using your router portal credentials (or via public key crypto, depending on configuration) before continuing.

> Note: If SSH will be left enabled after installation, disallow password login, enable brute force protection, and use public keys for login to enhance security.

##### Install Cloudflare DDNS Script
1. Log into your router via SSH, and navigate to `/jffs/scripts`.
2. Copy the `cloudflare_ddns` and `.cloudflare.example` files to that directory.
3. Rename `.cloudflare.example` to `.cloudflare`.
4. Edit `.cloudflare` with your Cloudflare email, API key, and zone ID from your Cloudflare portal.
5. Run `chmod 700 cloudflare_ddns`.
6. Run `chmod 600 .cloudflare`.
7. Run `./cloudflare_ddns list`.
8. Step 7 should have resulted in the creation of a log file named `cloudflare_ddns.log`. Open the log file and review the JSON response object, which should be a listing of your Cloudflare DNS records for the zone ID specified in Step 4.
> Note: If there is an error in the log file or no log file is present, ensure permissions are correct and that the text of the script is copied accurately. Double-check your credentials. If the error is from Cloudflare, you can review the text of the error in the JSON response and look any error code online.
9. Edit `.cloudflare` with the DNS record information (i.e. ID, name and type) obtained from Step 8. Ensure your text matches exactly.
10. Run `./cloudflare_ddns 1.1.1.1`.
11. Review the log file for the result of the last execution. If you see a successful response, verify against the Cloudflare portal. Otherwise, review the errors and correct as necessary.
> Note: You may get a throttled response if you have queried too quickly after Step 7. The script rate-limits to one query every 5 minutes. This is configurable in the `cloudflare_ddns` script or you can simply wait.
12. Ensure rate-limiting is working as expected by re-issuing the command in Step 10 a couple of times in quick succession and verifying that the log file shows frequent invocations are throttled.
13. If Steps 11 and 12 were successful, run `ln -s cloudflare_ddns ddns-start`. This creates a symbolic link with the name expected by the router firmware.
##### Enable Custom DDNS
In the router portal, under WAN -> DDNS,
- Enable DDNS client: Yes
- Server: Custom
- Host name: <the DNS host name your using in Cloudflare>
- HTTPS/SSL Certificate: None

Save the configuration.

##### Verification
If all is configured correctly, you should see:
1. A "successful" message on the router portal on saving the configuration. I believe this is determined by the `/sbin/ddns_custom_updated` commands being called properly within the `cloudflare_ddns` script.
2. In the router portal, under System Log, you should see an entries as below.
```
Nov  5 06:57:14 start_ddns: update CUSTOM , wan_unit 0
Nov  5 06:57:14 custom_script: Running /jffs/scripts/ddns-start (args: 73.57.185.52 ) - max timeout = 120s
Nov  5 06:57:16 ddns: Completed custom ddns update
```
3. A new log file for the script should have been created in a /tmp folder and it should contain a successful log entry. Find the log file by running `find / -name ddns-start.log 2>&1`.
4. The Cloudflare portal should reflect the updated public IP address of your router.

> Note: If any errors occur, review the router log file and the script log file for an indication of the error or manually re-run `./cloudflare_ddns list` and `./cloudflare_ddns 1.1.1.1` to identify and troubleshoot.

##### Clean Up
Once everything is configured and working properly, you may delete the `cloudflare_ddns.log` file from the `/jffs/scripts/` directory on the router. If SSH access is no longer needed, disable SSH on the router portal for security (especially if password authentication was used).
