# opnsense-porkbun-ddclient-fix
### Fix for the OPNsense DDNS ddclient plugin's Porkbun API

###### [Update 2025-01-17]: 
###### I updated my OPNsense firewall to v24.7.12 and it looks like ddclient was updated too, v3.11.2_2. I'd chosen not to lock the ddclient package in OPNsense so I'd see these updates and their effects. After the update, my logs showed ddclinet is using the new API URL but reverted to the wrong endpoint `/domain/A/subdomain` format. So we see the Porkbun plugin API endpoints fail, using the wrong format: `https://api.porkbun.com/api/json/v3/dns/retrieveByNameType/com/A/domain `. I reapplied the fixes below and my plugin is working again!

##### Disclaimer: The configurations and suggestions provided are for informational purposes only and are used at your own risk. I am not liable for any damage, data loss, or security issues resulting from their implementation. Please note that I am not affiliated with OPNsense or Porkbun in any way.


#### [DEC 2024]:
Porkbun updated thier API URL but the ddclient plugin had not been updated yet. People had suggested using the caddy plugin since it has updated its Porkbun parts but I didn't feel like changing yet. So, until the plugin is updated fully, you can update the `ddclient` perl script yourself.

Updating the API URL was pretty simple. The next problem was the ddclient plugin Porkbun code was using incorrect **editByNameType** URI endpoints (newest docs here: `https://porkbun.com/api/json/v3/documentation`). I'm not sure if Porkbun's API call formating was changed at some point but there's lots of threads online about this plugin's issues handling Porkbun subdomains before the URL change. The proper fix will should use the `on-root-domain` config from the original script but I havent had time for that yet.

#### This is the correctly formatted `retrieveByNameType` API Endpoint: `/api/json/v3/dns/retrieveByNameType/domain.com/A/subdomain`

**Note:** Multiple subdomains are simply stacked. For exmaple, the URI Endpoint for host: `sub2.sub1.domain.com` would be:

`https://api.porkbun.com/api/json/v3/dns/retrieveByNameType/domain.com/A/sub2.sub1`


The 'on-root-domain' logic part of the script to organize the endpoint doesnt seem to be working. It's not using my `domain.com` correctly in the endpoint. You could comment out the `if ($config{$host}{'on-root-domain'})' block below in Step 11, replace with these two lines, and call it a day if you don't use subdomains:

```
    $sub_domain = '';
    $domain = $host;
```

###  Here are the steps to update the API URL and also add logic to split traditional TLD subdomains for properly formatted endpoint calls. I understand the regex in my new split function doesnt work for domains like domain.it.io where the top-level domain (TLD) is split by a period (.it.io). 

1. First we're going to enable ssh access. I usually have this disbaled unless I **have** to access the shell. Go to `System > Settings > Administration`. Check **Enable Secure Shell**
2. And also check **Permit password login**
3. Click **Save** at the bottom of the page
   
![image](https://github.com/user-attachments/assets/7dfd4366-1106-465b-a7ac-0247ba8dcd98)


4. Log into your opnsense through your favorite terminal as a **non-root** user
5. Switch to root, **run:** `su root`
6. Press **8** for **Shell** at the OPNsense menu and hit enter
7. Add `api` to two URLs in ddclient around line 7113 and 7167,

   **run:** `sed -i -e 's|$url = \"https://porkbun.com/api|$url = \"https://api.porkbun.com/api|g' /usr/local/sbin/ddclient`
8. Now let's edit `ddclient`, **run:** `nano /usr/local/sbin/ddclient`  (`pkg install nano` if you don't have it installed)
9. Press `Ctrl + F` and paste in this to search: `nic_porkbun_update` then enter

10. Press `Ctrl + F` and enter once more should get you to the Porkbun section around line 7090. Directly below are two `foreach` loops. Inside them around Line 7104 is an `if` block. Comment or delete the `if` and `else` lines and parantheses.

11. Find the `if` block starting with: `if ($config{$host}{'on-root-domain'}) {` 

![image](https://github.com/user-attachments/assets/aec62aec-6f51-47e7-ba11-49370c404ef4)

12. Remove or comment that block and in its place, put this block.
##### The regex is basically looking for domains with more than one (1) period and splitting them. Feel free to remove the info debug lines.
```
            my $dot_count = ($host =~ tr/.//);
            if ($dot_count < 1) {
                  info("debug: less than 1 dot found in host name");
                  $sub_domain = '';
                  $domain = $host;
            }
            if ($dot_count > 1) {
                  info("debug: There are one or more subdomains in %s", $host);
                  ($sub_domain, $domain) = split(/\.(?=[^.]+\.[^.]+$)/, $host, 2);
            } else {
                  info("debug: There is no subdomain in %s", $host);
                  $sub_domain = '';
                  $domain = $host;
            }
```
![image](https://github.com/user-attachments/assets/4a5617ef-b402-44ce-ac11-273f5e256ff5)

13. Done. `Ctrl + X`, then `Y` to save, then enter
14. Now, go to your OPNsense web GUI and **Disable secure shell login!!**
15. After ssh is disbaled, go to `Services > Dynamic DNS > Settings` and click the **+** to add a server
16. Click to `Enable this virtual server`, give it a `description`, enter your `username` and `password` (these are the pk1_ and sk1_ key values you get from Porkbun), enter your hostname(s) to update, we'll use the `Interface` **IP check method**, `Interface` is **WAN**, **Enable** `Force SSL`, and then click **Save**

![image](https://github.com/user-attachments/assets/47316c09-e18b-4e98-a39f-8741a96dd432)


17. Then click **Apply** at the bottom of the page.



## And to verify it's all working:

1) You'll need active current Porkbun A Type DNS Records

![image](https://github.com/user-attachments/assets/0570c5e3-dbe3-4758-8c8f-360d60e89e36)

2) Probably should have stopped the service before we started, but go ahead and restart it now:

![image](https://github.com/user-attachments/assets/13ba2c2a-2b32-4ccb-85b5-914b30b36f56)



3) Now check the Logs:

![image](https://github.com/user-attachments/assets/18fb8b0d-7a00-4726-8785-d1a28d5d6299)
###### We Have SUCCESSES and FAILS where we'd expect. Success where there's actually a DNS record to update.
```
Notice	ddclient	 SENDING:  Curl system cmd to https://api.porkbun.com

Notice	ddclient	 UPDATE:   updating testdomain.com
Notice	ddclient	 INFO:     setting ipv4 address to 71.X.X.X for testdomain.com
Notice	ddclient	 INFO:     debug: There is no subdomain in testdomain.com
Notice	ddclient	 SENDING:  url="https://api.porkbun.com/api/json/v3/dns/retrieveByNameType/testdomain.com/A/"
Notice	ddclient	 SUCCESS:  updating ipv4: skipped: testdomain.com address was already set to 71.X.X.X.

Notice	ddclient	 UPDATE:   updating sub1.testdomain.com
Notice	ddclient	 INFO:     setting ipv4 address to 71.X.X.X for sub1.testdomain.com
Notice	ddclient	 INFO:     debug: There are one or more subdomains in sub1.testdomain.com
Notice	ddclient	 SENDING:  url="https://api.porkbun.com/api/json/v3/dns/retrieveByNameType/testdomain.com/A/sub1"
Notice	ddclient	 SUCCESS:  updating ipv4: skipped: sub1.testdomain.com address was already set to 71.X.X.X.

Notice	ddclient	 UPDATE:   updating sub2.sub1.testdomain.com
Notice	ddclient	 INFO:     setting ipv4 address to 71.X.X.X for sub2.sub1.testdomain.com
Notice	ddclient	 INFO:     debug: There are one or more subdomains in sub2.sub1.testdomain.com
Notice	ddclient	 SENDING:  url="https://api.porkbun.com/api/json/v3/dns/retrieveByNameType/testdomain.com/A/sub2.sub1"
Notice	ddclient	 FAILED:   updating sub2.sub1.testdomain.com: No applicable existing records.  #### Failed to update as expected, since I dont have a sub2.sub1.testdomain.com DNS A Record ####

Notice	ddclient	 UPDATE:   updating testdomain.com
Notice	ddclient	 INFO:     setting ipv4 address to 71.X.X.X for testdomain.com
Notice	ddclient	 INFO:     debug: There is no subdomain in testdomain.com
Notice	ddclient	 INFO:     forcing updating sub3.sub2.sub1.testdomain.com because no cached entry exists.
Notice	ddclient	 SENDING:  url="https://api.porkbun.com/api/json/v3/dns/retrieveByNameType/testdomain.com/A/sub3.sub2.sub1"
Notice	ddclient	 SUCCESS:  updating ipv4: skipped: sub3.sub2.sub1.testdomain.com address was already set to 71.X.X.X.
```







