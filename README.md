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
7) Add 'api' to two URLs in ddclient around line 7113:
 
    ![image](https://github.com/user-attachments/assets/bd2ab80c-3756-4829-b3f3-cd2ca92e1e13)
   and 7167:
   
   ![image](https://github.com/user-attachments/assets/7b81b571-1165-44b2-b296-d3159cff3f17)
run: `sed -i -e 's|$url = \"https://porkbun.com/api|$url = \"https://api.porkbun.com/api|g' /usr/local/sbin/ddclient`
10) Now let's edit `ddclient`, run: `nano /usr/local/sbin/ddclient`  (`pkg install nano` if you don't have it installed)
11) 


