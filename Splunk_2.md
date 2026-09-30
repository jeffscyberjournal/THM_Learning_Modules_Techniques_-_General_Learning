# Splunk 2

**BOTSv2 Dataset:**

BOTSv2 is a realistic Splunk security dataset containing Windows endpoint logs, Sysmon events, firewall data, network traffic, and IDS alerts. It is used to practice SOC investigations, threat hunting, incident response, and SPL query analysis within Splunk.

Task 1: Run from attack box or VM.

## Task 2 Dive into the Data

In this scenario, you take the role of Alice Bluebird, a security analyst assisting Frothly with investigating security incidents using Splunk.

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

## Task 3 100 Series Questions

Scenario: Investigate Amber Turing's communications with a potential competitor and identify the website visited, emails exchanged, contacts involved, and files sent.

### Q1: Identify the Competitor Website

1. Find Amber's IP Address

Search for references to Amber:
```
index="botsv2" amber
```

Focus on PAN traffic logs:
```
index="botsv2" sourcetype="pan:traffic"
```
Output: Amber's IP address.

**NOTE:** `pan:traffic` is the Palo Alto Networks firewall traffic log sourcetype. It contains network connection information recorded by a Palo Alto Networks firewall, including the usual source and destination IP addresses, ports, applications, actions, and web traffic details.

Those logs are ingested into Splunk as:
```
sourcetype=pan:traffic
```



### 2. Review Amber's HTTP Activity

Replace IPADDR with Amber's IP address:
```
index="botsv2" IPADDR sourcetype="stream:HTTP"
```
Output: HTTP requests made by Amber.

### 3. List Unique Websites Visited

Remove duplicates and display sites:
```index="botsv2" IPADDR sourcetype="stream:HTTP"
| dedup site
| table site
```
Output:
```
sitewebsite1
website2
...
```
The competitor's domain should stand out based on Frothly's industry.

Alternative: Filter by Industry

Use an industry-related keyword:

```
index="botsv2" IPADDR sourcetype="stream:HTTP" *INDUSTRY*
| dedup site
| table site
```
Output: Typically narrows results to the competitor website.

### 4. Focus on Traffic to the Competitor Website

```
index="botsv2" IPADDR sourcetype="stream:HTTP" COMPETITOR_WEBSITE
```
Output: HTTP activity between Amber and the competitor's website.

Use table to extract useful fields:

```
index="botsv2" IPADDR sourcetype="stream:HTTP" COMPETITOR_WEBSITE
| table uri uri_path site
```
Output: URLs visited, including pages containing executive contact information.

### 5. Investigate Email Communications

Identify Amber's email from previous results and use it for SMTP searches.

```
index="botsv2" sourcetype="stream:smtp" AMBERS_EMAIL COMPETITOR_WEBSITE
```
Output: Email exchanges between Amber and competitor personnel.

Useful fields may include:
```
| table sender recipient subject attachment
```

This assumes the fields are available for selection within the events. Splunk does not allow you to select these fields directly. Instead, you must use "Extract New" from the Interesting Fields panel and choose a sample event that contains the desired field. You can then create the extraction using either a regular expression (regex) or delimiter-based extraction. In this case, comma-delimited extraction works correctly.


### Lab Question Answers

**Q1 Amber Turing was hoping for Frothly to be acquired by a potential competitor which fell through, but visited their website to find contact information for their executive team. What is the website domain that she visited?**

search using 
```
index="botsv2"  amber  sourcetype="pan:traffic"
```
There is only 1 IP for src_ip or client_ip likely related to amber: 10.0.2.101

Using that with further search:
```
index="botsv2" 10.0.2.101 sourcetype="stream:HTTP" *beer*
```
Add dedup site to filter further
```
index="botsv2" 10.0.2.101 sourcetype="stream:HTTP" *beer* 
| table site 
| dedup site
```
leads to just one site: www.berkbeer.com




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



**Q3 What is the CEO's name? Provide the first and last name.**

```
index="botsv2" sourcetype="stream:smtp" amber
```
Filters down 26 smtp related events linked to word amber.

For what ever reason regex does not seem to work well with filtering, the image in previous question however is called ceoberk which is a clue worth trying. Using berk instead of amber leads to just one event with smtp making sense its from sender 

```
index="botsv2" sourcetype="stream:smtp" berk
```
- From content_body section in the line "Give me a call this afternoon if you= are free.=C2=A0=0A=0AMartin Berk..."
```
     --=_8177b74425496b166cbde61bd37bbf96

     Hello Amber,=C2=A0=0A=0AGreat to hear from you, yes it is unfortunate th=
e way things turned=0Aout. It would be great to speak with you directly,=
 I would also like=0Ato have Bernhard on the call as I think he might ha=
ve some questions=0Afor you. =C2=A0Give me a call this afternoon if you=
 are free.=C2=A0=0A=0AMartin Berk=0ACEO=0A777.222.8765=0Amberk@berkbeer.=
com=0A=0A----- Original Message -----=0AFrom: "Amber Turing" <aturing@fr=
oth.ly>=0ATo:"mberk@berkbeer.com" <mberk@berkbeer.com>=0ACc:=0ASent:Fri,=
 11 Aug 2017 15:49:01 +0000=0ASubject:Amber from Froth.ly=0A=0A=09Mr. Be=
rnhard,=0A=0A=09=C2=A0=C2=A0 I was very sorry to hear about the acquisit=
ion falling through.=0AI was very excited to work with you in the future=
.. I have to admit, I=0Aam a little worried about my future here. I=E2=80=
=99d love to talk to you=0Aabout some information I have regarding my wo=
rk.=0A=0A Amber Turing=0A Principal Scientist=0A 867.322.1123=0A Froth.l=
y=0A=0A=09

     <html><body style=3D"font-family: Helvetica,Arial,sans-serif; font-size:=
 12px;">Hello Amber,=C2=A0<div><br></div><div>Great to hear from you, ye=
s it is unfortunate the way things turned out. It would be great to spea=
k with you directly, I would also like to have Bernhard on the call as I=
 think he might have some questions for you. =C2=A0Give me a call this a=
fternoon if you are free.=C2=A0</div><div><br></div><div>Martin Berk</di=
v><div>CEO</div><div>777.222.8765</div><div>mberk@berkbeer.com<br><br><b=
lockquote class=3D"atmailquote"><br>----- Original Message -----<br><div=
 id=3D"origionalMessageFromField" style=3D"width:100%;display:inline;bac=
kground:rgb(228,228,228);"><div style=3D"display:inline;font-weight:bold=
;">From:</div> "Amber Turing" &lt;aturing@froth.ly&gt;</div><br><div id=
=3D"origionalMessageToField" style=3D"display:inline;font-weight:bold;">=
To:</div>"mberk@berkbeer.com" &lt;mberk@berkbeer.com&gt;<br><div id=3D"o=
rigionalMessageSentField" style=3D"display:inline;font-weight:bold;">Cc:=
</div><br><div style=3D"display:inline;font-weight:bold;">Sent:</div>Fri=
, 11 Aug 2017 15:49:01 +0000<br><div id=3D"origionalMessageSubjectField"=
 style=3D"display:inline;font-weight:bold;">Subject:</div>Amber from Fro=
th.ly<br><br><br><div class=3D"WordSection1">=0A<p class=3D"MsoNormal">M=
r. Bernhard,</p><p></p>=0A<p class=3D"MsoNormal">=C2=A0=C2=A0 I was very=
 sorry to hear about the acquisition falling through. I was very excited=
 to work with you in the future. I have to admit, I am a little worried=
 about my future here. I=E2=80=99d love to talk to you about some inform=
ation I have regarding=0A my work.<br><br>=0AAmber Turing<br>=0APrincipa=
l Scientist<br>=0A867.322.1123<br>=0AFroth.ly</p><p></p>=0A</div>=0A=0A=
=09</blockquote></div></body></html>
```


**Q4 What is the CEO's email address?**

This is in the same content_body or sender, sender_email section of event.

"mberk@berkbeer.com"



**Q5 After the initial contact with the CEO, Amber contacted another employee at this competitor. What is that employee's email address?**

Another employee email address assuming same domain @berkbeer in place of berk in previous search, 6 results found:
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

Email sent at 8/30/17 3:08:00.075 PM from Amber contains the attackment:
```
 attach_filename: [ [-]
     Saccharomyces_cerevisiae_patent.docx 
```

**Q7 What is Amber's personal email address?**

The last email sent from Amber in the contain_body section there is base64 code, in it is a section:

```
...
<p class="MsoNormal">Thanks for taking the time today, As discussed here is the document I was referring to.&nbsp; Probably better to take this offline. Email me from now on at
<a href="mailto:ambersthebest@yeastiebeastie.com">ambersthebest@yeastiebeastie.com</a>
...
```


## 200 Series Questions

### Question 1: TOR Version Installed by Amber

Start with a keyword search:
```
index="botsv2" amber tor
```
This returns 325 events. Reverse the event order and add another keyword to narrow the results.

Command
```
index="botsv2" amber tor KEYWORD
```

Replace KEYWORD with a term that helps identify the TOR installation version.

Trying KEYWORD 'install' shows "C:\Users\amber.turing\Downloads\torbrowser-install-7.0.4_en-US.exe" in 125 events. Less successful keywords were browser (325), exe (325)

### Questions 2 & 3: BrewerTalk IPs

Determine:

- The public IP address of brewertalk.com
- The IP address performing a web vulnerability scan against it

Use the techniques from previous searches.

Try
```
index="botsv2" "brewertalk.com"
```
Then try field site leading to 4 DNS url and closer look leads to 4 sites

Then use field dest_ip and search external (2 out of the 10), leading to:
```
index="botsv2" "brewertalk.com" dest_ip="52.40.10.231"
```
Shows 
```
Values 	                                             Count 	%
www.brewertalk.com 	                                   9,860 	99.026% 	
brewertalk.com                                         86        0.864% 	
ec2-52-40-10-231.us-west-2.compute.amazonaws.com:8088 	10        0.1% 	
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
   set_cookie: [ [+]
   ]
   site: www.brewertalk.com
   src_ip: 52.40.10.231
   src_mac: 0A:96:DA:8D:C8:A1
   src_port: 57245
   status: 200 
```
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
uri_path       count
/member.php	662
/search.php	164
/	          47
/admin/	     6
```


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
taking a closer look at the data field contains:
```
regcheck1=&regcheck2=true&username=makman&password=mukarram&password2=mukarram&email=mak@live.com&email2=mak@live.com&referrername=&imagestring=F7yR4&imagehash=1c1d0e6eae9c113f4ff65339e4b3079c&answer=4&allownotices=1&receivepms=1&pmnotice=1&subscriptionmethod=0&timezoneoffset=0&dstcorrection=2&regtime=1416039333&step=registration&action=do_register&regsubmit=Submit Registration!&question_id=makman' and updatexml(NULL,concat (0x3a,(SUBSTRING((SELECT password FROM mybb_users ORDER BY UID LIMIT 5,1), 32, 31))),NULL) and '1
```
That payload is an error-based SQL injection attack.

The key indicator is:
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

Example:

```
UPDATEXML(NULL, CONCAT(':', database()), NULL)
```

Might produce an error similar to:

```
XPATH syntax error: ':mydatabase'
``
The attacker then reads the data from the error message.

### What this payload is doing

The injected sql string:

```
' and updatexml(...)
and '1
```

Attempts to append a condition to the original query.

Inside the UPDATEXML() call:

```
SELECT password
FROM mybb_users
ORDER BY UID
LIMIT 5,1
```

retrieves the password hash from the 6th user record (offset 5).

Then:

```
SUBSTRING(..., 32, 31)
```

extracts part of that hash.

Then:

```
CONCAT(0x3a, ...)
```

Adds a colon (:) before the extracted data.

Finally, UPDATEXML() causes an error containing that value, allowing the attacker to exfiltrate the password hash through the application's error output.



Questions 6 & 7: XSS Cookie and Spear-Phishing User

Start by gathering information about Kevin.

Command

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" kevin
Show more lines

Identify Kevin's full name, then investigate the XSS attack.

Focus on:

Kevin's HTTP traffic
XSS payloads
Cookie values sent to a malicious URL

Once you find the relevant events, determine the username created through a spear-phishing attack.

A keyword search should help:

Command

Plain Text
spl isn’t fully supported. Syntax highlighting is based on Plain Text.
index="botsv2" KEYWORD
Show more lines

Replace KEYWORD with an appropriate search term.

Questions to Answer
What version of TOR Browser did Amber install to obfuscate her web browsing?
What is the public IPv4 address of the server running www.brewertalk.com?
What IP address performed a web vulnerability scan against www.brewertalk.com?
What URI path was attacked from the IP address in Question 3? (Include the leading /.)
What SQL function was abused on that URI path?
What cookie value did Kevin's browser transmit during the XSS attack? (Digits only.)
What brewertalk.com username was maliciously created through a spear-phishing attack?
