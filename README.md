8.7. MISSING X-CONTENT-TYPE-OPTIONS HEADER
Status: Open (Previously Discovered)
Rating: Low
Location:
• https://sadrd.manulife.com/app/
Description:
The 'X Content Type Options' response header tells web browsers to disable MIME and content
sniffing.
Previous Proof of Concept:
1. Configure the browser to use a proxy like Burp Suite.
2. Launch the application in the browser.
3. Intercept the request in Burp suite and send it to repeater.
4. Navigate to repeater, click on send button and observe the application response.
5. It can be observed that the "X-Content type options " Header is missing from the
response.
Retest Proof of Concept:
1. Open the Burp Suite and its pre-built browser.
2. Navigate the specified URL in the browser.
3. Right-click and select "Inspect," then go to the "Network" tab and refresh the page.
In Conclusion: After investigating the entire application, it has been verified that the previously
discovered vulnerability, 'Missing X-Content-Type-Options Header,' remains open because not
all endpoints have the specified header in the production environment.
Recommended Remediation:
The only valid for this header is “X-Content-Type-Options: nosniff” 
