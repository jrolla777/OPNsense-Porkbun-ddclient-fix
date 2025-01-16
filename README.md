# opnsense-porkbun-ddclient-fix
Fix for the OPNsense Dynamic DNS service ddclient Porkbun API plugin

Porkbun updated thier API URL but the Dynamic DNS plugin for OPNsense using ddlcient as the backend, has not been updated. So i searched and found the ddclient perl script at **/usr/local/sbin/ddclient**. It's been a long time since I worked with perl but thankfully it wasn't too complicated :) I worked on these steps over a couple afternoons so this is still very much a work in progress. 

First, we need to update the API URL. That's easy, simple **sed** command to substitue in two lines. The next problem, which I'm not going to spend much time elaborating on now, the ddclient plugin Porkbun code was using **editByNameType** calls that was failing (https://porkbun.com/api/json/v3/documentation). The format needed to be changed and the 'on-root-domain' part seemed unnecessary. You could comment out the `if` block from Step 12 and simply replace with these two lines and the plugin will work if you don't need to update any sub-domains (ex. `sub2.sub1.domain.com`) 

```
    $sub_domain = '';
    $domain = $host;
```

1. First we're going to enable ssh access. I usually have this disbaled unless I **have** to access the shell. Go to `System > Settings > Administration`. Check **Enable Secure Shell**
2. And also check **Permit password login**
3. Click **Save** at the bottom of the page
   
![image](https://github.com/user-attachments/assets/7dfd4366-1106-465b-a7ac-0247ba8dcd98)


4. Log into your opnsense through your favorite terminal as a **non-root** user
5. Log in as root, **run:** `su root`
6. Press **8** for Shell and hit enter
7. Add 'api' to two URLs in ddclient around line 7113 and 7167,

   **run:** `sed -i -e 's|$url = \"https://porkbun.com/api|$url = \"https://api.porkbun.com/api|g' /usr/local/sbin/ddclient`
8. Now let's edit `ddclient`, **run:** `nano /usr/local/sbin/ddclient`  (`pkg install nano` if you don't have it installed)
9. Press `Ctrl + F` and paste in this to search: `nic_porkbun_update` then enter

10. Press `Ctrl + F` and enter once more should get you to the Porkbun section around line 7090. Directly below are two `foreach` loops. Inside them around Line 7104 is an `if` block. Comment or delete the `if` and `else` lines and parantheses.

Starting at: `if ($config{$host}{'on-root-domain'}) {`

![image](https://github.com/user-attachments/assets/aec62aec-6f51-47e7-ba11-49370c404ef4)


11. In its place, put this block:
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

12. Done. `Ctrl + X`, then `Y` to save, then enter
13. Now, go to your OPNsense web GUI and **Disable secure shell login!!**
14. After ssh is disbaled, go to `Services > Dynamic DNS > Settings` and click the **+** to add a server
15. Click to `Enable this virtual server`, give it a `description`, enter your `username` and `password` (these are the pk1_ and sk1_ key values you get from Porkbun), enter your hostname(s) to update, we'll use the `Interface` **IP check method**, `Interface` is **WAN**, **Enable** Force SSL, and then click **Save**

![image](https://github.com/user-attachments/assets/7859de43-9d45-4153-9acd-038cdd614ad6)

16. Then click **Apply** at the bottom of the page.







