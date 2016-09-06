---
layout: post
title:  "Analyzing Salary Data from Reddit for System Administration"
date:   2016-09-05
categories: career system administration scripting ruby
---

/r/sysadmin on Reddit had an [interesting post](https://www.reddit.com/r/sysadmin/comments/516jbk/rsysadmin_survey_on_salary_location_job/) come up with a link asking everyone to anonymously share their salary and some other information. The original poster was also nice enough to share the results in a google doc. With some free time I thought it would be interesting to do some rough analysis of the data and see if I found anything interesting. 

The columns are title, salary, benefits, years of experience and job responsibilities. 
The original form was all text so it was an interesting challenge to parse, but I think I managed. This also makes the data not exact, but I think with the amount of responses the occasional error is ok. All together there about 740 responses when I wrote this. 

First thing was cleaning up the data and deciding what to use. I decided to ignore the benefits columns completely. The actual value of benefits is somewhat personal and hard to compare. I also converted all of the answers given into base yearly compensation in USD. I ignored any extra information like bonus percentage and stocks. The currencies I converted were AUD, EURO, CAD, KR and CAD. The answers that were given by hour were multiplied by 40 hours then 52 weeks. Experience was the next column to clean up. I took the first number given and used that, since there was a lot of extra information written. There were also a lot responses like "10+". These became 10, something to remember when looking at the following experience data.

**Some Data:**

To start I ended up with 687 valid salary records. For whatever reason there was a problem with trying to parse it. I also didn't count anything that was under 10,000. I assumed anything that low was an error.
I had 654 valid experience entries.

Average Salary: $70,079 USD  *(Everything is USD from now on)*  
Average years of experience: 7 years

Not a bad average. An average on it's own doesn't have enough information.   
We'll graph the salaries and to keep the theme of systems administration we'll have a command line output

I used 20 slots, and calculated a range based on my minimum of 10,000 and the maximum salary response.
I ended up with intervals of 10304. 

**Salaries Graphed** *(may need to scroll)*

~~~	
0 (%0.0)	[
10304 (%2.04)	[xxxxxxxxxxxxxx
20608 (%4.8)	[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
30912 (%13.54)	[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
41216 (%13.1)	[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
51520 (%14.12)	[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
61824 (%12.81)	[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
72128 (%10.19)	[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
82432 (%8.88)	[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
92736 (%5.82)	[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
103040 (%3.93)	[xxxxxxxxxxxxxxxxxxxxxxxxxxx
113344 (%3.06)	[xxxxxxxxxxxxxxxxxxxxx
123648 (%2.18)	[xxxxxxxxxxxxxxx
133952 (%1.46)	[xxxxxxxxxx
144256 (%1.6)	[xxxxxxxxxxx
154560 (%0.58)	[xxxx
164864 (%0.58)	[xxxx
175168 (%0.29)	[xx
185472 (%0.0)	[
195776 (%0.44)	[xxx
206080 (%0.29)	[xx
216384 (%0.0)	[
226688 (%0.15)	[x
236992 (%0.15)	[x
247296 (%0.0)	[
~~~

Not anything I would consider an extreme outlier. 

The next interesting thing would be  
**The top 10 Salaries**

|Salary| Title | Experience | Location |
|----+----|
|165000 | Presales Sr. SE | 15 | South, USA |
|175500 | Director (Technical) | 20 | UK |
|185000 | Cloud operations engineer | 16 | Sf bay area |
|200000 | Systems Engineer | 3 | Bay Area, California |
|200000 | Linux Systems Administrator | 20 | San Francisco |
|204000 | Principal Engineer | 25 | SF Bay Area |
|208000 | Principal Network Architect | 9 | Texas |
|213000 | CTO | 16 | NYC |
|235000 | Director of Cloud Operations  | 3 | New york |
|247000 | CTO | 9 | North West England, UK |


SF bay area takes 4 spots here. UK takes 2 which is interesting.  NYC takes two spots.

Looking at how salary correlates with experience  
There wasn't too many entries by people with more then 15 years of experience.

**Represented years of experience**

*Lowest*

|Experience|Percentage|Count|
|----+----|
| 23 | %0.15 | 1|
| 31 | %0.15 | 1|
| 26 | %0.15 | 1|
| 28 | %0.15 | 1|
| 19 | %0.29 | 2|


*Highest*

|Experience|Percentage|Count|
|----+----|
| 4 | %8.73 | 60|
| 2 | %9.61 | 66|
| 5 | %10.77 | 74|
| 3 | %11.06 | 76|
| 10 | %11.35 | 78|

I think "10" is overrepresented by "10+" responses and rounding

**Experience vs Average Salary**  
For this I averaged the salary for each response grouped by the years of experience given.

*Lowest*

|Experience|Salary|Count|
|----+----|
| 1 | 45212| 27 |
| 0 | 48316| 26 |
| 4 | 52094| 60 |
| 3 | 54167| 76 |
| 2 | 54890| 66 |


*Highest*

|Experience|Salary|Count|
|----+----|
| 18 | 118166| 9|
| 21 | 121000| 3|
| 19 | 122045| 2|
| 23 | 129000| 1|
| 28 | 150000| 1|

There is a positive correlation between experience and salary. I don't think this is unexpected. Although it would have been nice to have more responses by people with more experience.


**Engineer vs Admin**  

I thought it would be interesting to see if "engineers" are paid more then "administrators". I excluded managers and directors from this.

197 "Engineers"  
222 "Admins"  
$80,938 - Average engineer salary  
$62,944 - Average admin salary  

**How much more do managers make then other employees?**  

49 responses were categorized as manager or director.

Manager Average: $87,161  
Engineer average : $80,938  
Employee(Admin+Engineer) average : $71,404  


**Has "senior" been watered down?**  
The lowest amount of experience for a senior title was 3 years with 7 responses. This was actually in the top five for counting the responses based on years of experience.  
10,8,7,3,15 years  
Looks like overall senior does imply experience. 

**Windows vs Linux**  
There weren't a lot of entries that included the tech they work with.

Linux total: 76  
Windows total: 87  
Linux average salary: $73,102  
Windows average salary: $65,704  

**SF Bay Area vs London**  
There was some discussion in the original thread about The SF Bay Area and London.

11 SF Bay Area Responses  
13 London Responses

$153,181	Average SF Bay Area Salary  
$52,796		Average London Salary

**SF Bay Area**

This is were I live so I was interested in it.

We already saw the average salary is $153,181. Comparable to programing salaries.

The lowest was $105,000 and the highest was $204,000. All of the responses except one were engineers. One was a director at the 3rd lowest salary for the region. 

The SF Bay Area also seems like a place for the more experienced. Half of the responses were over 10 years. The average for the region is 9 years. It doesn't look like these jobs are going to new grads.


The script is on my github if you would like to try it.  
[https://github.com/tejom/analysisscript](https://github.com/tejom/analysisscript)  
The original data  
[https://docs.google.com/spreadsheets/d/1aLQTZLVEKUgJ7FvCB1Jkyq7kXDgke2Av379xRdUor70/edit#gid=1567124098](https://docs.google.com/spreadsheets/d/1aLQTZLVEKUgJ7FvCB1Jkyq7kXDgke2Av379xRdUor70/edit#gid=1567124098)


