# THM - WebOSINT

#### Name: WebOSINT
#### Rating: Easy

----------------------------------------------------------------------

webosint.jpeg

#### Task 1 When A Website Does Not Exist

"Your job is to find as much information as you can about the website RepublicofKoffee.com."

Lets join the room and begin enumerating the website!

### Task 2 Whois Registration


###### Question 1:  What is the name of the company the domain was registered with? 

Lets use the whois tool from the command line to begin answering the questions.

```text
whois RepublicOfKoffee.com
```

q1.png

###### Question 2: What phone number is listed for the registration company?

q2.png

###### Question 3: What is the first nameserver listed for the site?

q3.png

###### Question 4: What is listed for the name of the registrant?

q4.png

###### Question 5: What country is listed for the registrant?

https://www.whoxy.com/republicofkoffee.com#history

q5.png

### Task 3 Ghosts of Websites Past

"Don't be discouraged when your initial searches on a website turn up empty.That's where Archive.org and the Internet Wayback Machine come into play."

###### Question 6: What is the first name of the blog's author? 

q6.png

###### Question 7: What city and country was the author writing from?

q7.png

###### Question 8: [Research] What is the name (in English) of the temple inside the National Park the author frequently visits?

q8a.png

q8b.png

### Task 4 Digging Into DNS

###### Question 9: What was RepublicOfKoffee.com's IP address as of October 2016?

q9.png

###### Question 10: Based on the other domains hosted on the same IP address, what kind of hosting service can we safely assume our target uses?

q10.png

###### Question 11: How many times has the IP address changed in the history of the domain?

q11.png

### Task 5 Taking Off The Training Wheels 

```text
"Congratulations on making it this far.

You'll need all of the skills you've learned so far for this task.

All I have for you, is a domain:

heat[dot]net

Good luck!"
```
###### Question 12:  What is the second nameserver listed for the domain? 

```text
whois heat.net
```

q12.png

###### Question 13: What IP address was the domain listed on as of December 2011?

q13.png

###### Question 14: Based on domains that share the same IP, what kind of hosting service is the domain owner using?

shared

q14.png

###### Question 15: On what date did was the site first captured by the internet archive? (MM/DD/YY format)

q15.png

###### Question 16: What is the first sentence of the first body paragraph from the final capture of 2001?

q16.png

###### Question 17: Using your search engine skills, what was the name of the company that was responsible for the original version of the site? 

q17.png

###### Question 18: What does the first header on the site on the last capture of 2010 say?

q18.png

### Task 6 Taking A Peek Under The Hood Of A Website

```text
"Isn't it kind of interesting how the website disappeared for a period of time and came back?

Clearly the purpose of the site is different now. Let's roll up our sleeves and figure out what's going on."
```

###### Question 19: How many internal links are in the text of the article? 

5

q19.png

###### Question 20: How many external links are in the text of the article? 

q20.png

###### Question 21: Website in the article's only external link ( that isn't an ad)

q21.png

###### Question 22: Try to find the Google Analytics code linked to the site

q22.png

###### Question 23: Is the the Google Analytics code in use on another website? Yay or nay

nay

###### Question 24: Does the link to this website have any obvious affiliate codes embedded with it? Yay or Nay

nay

### Task 7 Final Exam: Connect the Dots 

```text

"Experienced OSINT researchers will tell you that chasing rabbit holes all day and night without being able to make some solid connections is not OSINT.

OSINT refers to the patterns that start to emerge as we connect the dots in the analysis of the data. 

Congrats! you found that our target, heat[.]net, links to an interesting external site. A question remains though: Why???

There is no affiliate code in the link, so there is no obvious financial connection between the two. Maybe there's another kind of connection.

This is your final exam, and there is exactly one question.

Get busy!"
```

###### Question 23:  Use the tools in Task 4 to confirm the link between the two sites. Try hard to figure it out without the hint. 

q23a.png

q23b.png

q23c.png

Thanks for following along!

-Ryan