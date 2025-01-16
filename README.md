# opnsense-porkbun-ddclient-fix
Fix for the OPNsense Dynamic DNS service ddclient Porkbun API plugin

Porkbun updated thier API URL but the Dynamic DNS plugin for OPNsense using ddlcient as the backend, has not been updated. So i searched and found the ddclient perl script at **/usr/local/sbin/ddclient**. It's been a long time since I worked with perl but thankfully it wasn't too complicated :) I worked on these steps over a couple afternoons so this is still very much a work in progress. 

First, we need to update the API URL. That's easy, simple **sed** command to substitue in two lines. The next problem, which I'm not going to spend much time elaborating on now, the ddclient plugin Porkbun code was using **editByNameType** calls that was failing (https://porkbun.com/api/json/v3/documentation). The formate 

But, I noticed in my research, lots of people were having similar issues with porkbun. Lots of them had multiple sub-domains, for example: **sub2.sub1.domain.com**. 


1) First we're going to enable ssh access. I usually have this disbaled unless I **have** to access the shell. Check **Enable Secure Shell**
2) And also check **Permit password login**
3) Click **Save** at the bottom of the page
![image](https://github.com/user-attachments/assets/05147d4c-ac4b-43f7-a442-cd0ab1514e21)

4) Log into your opnsense through your favorite terminal as a **non-root** user
5) Log in as root, run: `su root`
6) Press **8** for Shell and hit enter
7) Add 'api' to two URLs in ddclient around line 7113 and 7167, run: `sed -i -e 's|$url = \"https://porkbun.com/api|$url = \"https://api.porkbun.com/api|g' /usr/local/sbin/ddclient`
10) Now let's edit `ddclient`, run: `nano /usr/local/sbin/ddclient`  (`pkg install nano` if you don't have it installed)
11) Press `Ctrl + F` and paste in this to search: `nic_porkbun_update` then enter
12) Press `Ctrl + F` and enter once more should get you to aorund line 7090 in the Porkbun section. Directly below are two `foreach` loops. Inside around Line 7104 is an `if` block. Comment or delete the `if` and `else` lines and parantheses.

Started at: `if ($config{$host}{'on-root-domain'}) {`
![image](https://github.com/user-attachments/assets/84d8ee25-1883-4c45-8bdf-72d15e1f17bb)

15) In its place, put this block:
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
                  ## $rrset_type = "CNAME" ?
            } else {
                  info("debug: There is no subdomain in %s", $host);
                  $sub_domain = '';
                  $domain = $host;
              }
```
16) Done. `Ctrl + X`, then `Y` to save, then enter
17) Now, go to your OPNsense web GUI and **Disable secure shell login!!**
18) Go to `Services > Dynamic DNS > Settings` and click the **+** to add a server
19) Click to `Enable this virtual server`, give it a `description`, enter your `username` and `password` (these are the pk1_ and sk1_ key values you get from Porkbun), enter your hostname(s) to update, we'll use the `Interface` **IP check method**, `Interface` is **WAN**, **Enable** Force SSL, and then click **Save**

![image](https://github.com/user-attachments/assets/7859de43-9d45-4153-9acd-038cdd614ad6)
20) Then click Apply at the bottom of the page.







