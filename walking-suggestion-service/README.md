# Walking Suggestion Service
The walking suggestion service suggests if the heartsteps-server should send a walking-suggestion message to the participant.

## Installation and runtime
To run the walking-suggestion-service, the heartsteps_service-template image needs to be built first. The following commands will build the heartsteps_service-template and then run the walking-suggestion-service.

``` bash
docker-compose build service-template
docker-compose run --service-ports walking-suggestion-service
```

## Production data
There is a need to pull and test data that is being generated by the production walking-suggestion-service.

To copy production data from the google storage bucket to your local machine:
```bash
docker-compose run walking-suggestion-service copy-files
```

**List of changes made to HS Bandit**

-	Change the default probability to 0.1 on Sept 6
-	Update the prior distribution using HS V2 participants’ data on Sept 6
-	Change the default probability to 0.2 on Oct 30
-	Expand the availability definition on Oct 19



## 1. Initialization

### WHEN TO CALL

The *initialize* service is called for each participant at the end of the day when the participant has 
installed the HS app on the phone and specifies the five time slots.

For example, if the participant sets up Fitbit and registers with the HeartSteps website on Monday (day 0), then, 
the participant would get the text message to install the HeartSteps app the following Monday (day 7). If the 
participant installs HeartSteps on the same day (e.g. Monday, day 7), the *initialize* service would be called 
at the end of Monday (day 7), e.g. Tuesday morning at 1:30am.   On the other hand, if the participant does not 
set up the HeartSteps app on the same day but instead install on Tuesday (day 8) for example, the *initialize*
 service would be called at the end of Tuesday, that is, 1:30am on Wednesday

In the rest of the document, I call the time period before the app is installed (excluding the day in which 
the Fitbit is set up) as "the week with no app" (although it could be more than 7 days if the participant 
does not install the HS app on the same day they receive the text message). 


### INPUT-OUTPUT
The *initialize* service has no output except for a message indicating successful initialization.  
Below is an example of json input. 

~~~json
{
	"userID":1,
	"totalStepsArray":[12310,10000,null,10000,10000,11111,22222],
	"preStepsMatrix":[
		{"steps":[null,2,null,30,10]},
		{"steps":[123,243,1231,30,100]},
		{"steps":[103,1232,null,301,103]},
		{"steps":[100,2,null,30,null]},
		{"steps":[1100,23,null,303,100]},
		{"steps":[100,2,null,30,100]},
		{"steps":[100,2,null,30,100]}],
	"postStepsMatrix":[
		{"steps":[100,2,null,30,100]},
		{"steps":[100,2,null,3012,100]},
		{"steps":[null,21,null,330,1010]},
		{"steps":[null,2,0,30,null]},
		{"steps":[100,2,1,30,100]},
		{"steps":[100,2,15,30,100]},
		{"steps":[100,2,null,30,100]}]
}
~~~

1. `userID`: The user Id is the unique id assigned to a participant. This id is the same as the study id assigned to participants when they are enrolled in the HeartSteps study.

2. `totalStepsArray` 

  - A vector of the daily total steps that are tracked by Fitbit for each of the days in the week with no app. 
  Fitbit defines a day as the period from 12:00am to 11:59pm in the participant's local time. 
  The data reported here will be directly collected from 
  the [Fitbit API's daily activity summary](https://dev.fitbit.com/build/reference/web-api/activity/#get-daily-activity-summary). 
  The vector is ordered in chronological order, e.g. the first number corresponds to the total numbers of steps in the first day 
  after the day in which participant sets up the Fitbit (note that the step count on the day when the participant sets up the 
  Fitbit is not included), and the last number corresponds to the total numbers of steps in the day when the participant 
  installs the app and specifies the five time slots. 

  - The daily step count for a day will be marked as `null` if any of the followings holds:
  
  	(1) The participant is classfied as "not wear the sensor for the day"  based on the heart rate data. The heart rate data is the [minute level intraday heartrate data from the Fitbit Web API.](https://dev.fitbit.com/build/reference/web-api/heart-rate/#get-heart-rate-intraday-time-series) If there is any non-zero minute level heartrate for a participant, they will be counted as having wore their sensor that day. 
	
	(2) the step count can not be querired from Fitbit server (e.g. the user revokes the consent for data collection and
	thus the Fitbit server does not have the step count data)

   	 If neither of above is true, then the recieved input for the correspond day will be a number (including 0, meaning 
	that the pariticipant is classified as "wear the sensor" and the Fibit collects 0 step during the day)

3. `preStepsMatrix` and `postStepsMatrix`

  - `preStepsMatrix` is a matrix of step counts collected 30 min prior to each decision times during the the week with no app. 
  The first element corresponds to the five pre-treatment step counts (in chronological order) in the first day after the 
  participant sets up the Fitbit (note that the day when the participant sets up the Fitbit is not included).  The second 
  element corresponds to the five pre-treatment step counts (in chronological order) in the second day and so on.  
  Similarly, `postStepsMatrix` is for the step count 30 min after each decision time. 

  - The pre (post)- 30 min step count for a decision time will be marked as `null` if any of the followings holds:

  	  (1) the participant is classfied as "not wear the sensor during the 30 min window before (after) the decision time "  
	based on the heart rate data (same above)

   	 (2) the step count can not be querired from Fitbit server (e.g. the user revokes the consent for data collection 
	and thus the Fitbit server does not have the step count data)

  	  If neither of above is true, then the recieved input for the correspond decision time will be a number; including 0, 
	meaning that the pariticipant is classified as "wear the sensor" and the Fibit collects 0 step during the 30 min window before (after) the decision time.

4.pooling

5. date

6. delieverMatrix

7. prioAntiMatrix

### CONDITION CHECKING

The current version of *initialize* service requires the input satisfying the following conditions 
**(otherwise, the service will send Peng and Nick an email and/or write a warning with a dump of the data in a file; need to decide what to do then)**


1. [server] Include all the required information (e.g. user id, total steps array, pre and post steps matrix)

2. [server]The length of `totalStepsArray ` matches with the size of `preStepsMatrix ` and `postStepsMatrix `.

3. [data quality] At least one non-missing (e.g. not marked as `null`) record for the total step data over the week with no app.  
For each time slot, at least one non-missing record (e.g. not marked as `null`) for the pre-30 min step data  over the week 
with no app. For each time slot, at least one non-missing (e.g. not marked as `null`) record for the post-30 min step data  
over the week with no app.  For example, it does not allow the case where all of the daily total step count is missing.  




## 2. Decision Making

### WHEN TO CALL
The *decision* service is called for each participant at each of the decision times after the week with no app. 
It *must* be called at each of five decision times in a day (even if the participant is currently unavailable). 
*(It is possible that a technical server failure would stop the decision service from being called, but we hope to avoid that situation)*


### INPUT-OUTPUT
The *decision* service outputs a json file (see below for an example of the output)

~~~json
{
    "probability": 0.5,
    "send": true,
    "inTrial": 0,
    "type": 0
}
~~~

1. `userID`, the user ID (to ensure there is no user mismatch between the input and output)
2. `probability`, the randomization probability, 
3. `send`, the indicator of whether to send the activity message
4. `inTrial `, indicating the type of user:  `0` means if this is test user, `1` if user is formally in the trial, and `2` if the user is in the pooled bandit group of 10 people. 
5. `type`, indiciating how the probability is calculated: `0` means we are using the fixed randomization 
probability (e.g. 0.25 if available), `1` means the probabilility is calculated based on person specific bandit algorithm
 if available, `2` means the probabilility is calculated based on pooled bandit algorithm if available.  



Shown below is an example of json input for user `1` at decision time `2` on day `3`. 

~~~json
{
  	"userID": 1,
 	 "studyDay": 3,
 	 "decisionTime": 2,
 	 "availability": true,
 	 "priorAnti": false,
 	 "lastActivity": false,
 	 "location": 1,
	 "watch": true
}
~~~

1. `userID`: the user ID. 

2. `studyDay`: index of the current day in the study, starting at `1` on the first day (that is, the day after the participant installs the HS app on the phone)

3. `decisionTime`: index of the current time slot (i.e. decision time) in a day, ranging from `1` to `5`

4. `availability`: the indicator of availability at the current decision time (either `true` or `false`)

5. `priorAnti`

	- For the 1st decision time, `priorAnti` is the indicator of whether there is any anti-sedentary message delivered
	 to user's phone between the "start of the day" and the 1st decision time. The "start of day" here is specified 
	 in the anti-sedentary message scheduling. If a participant sets their first decision time to before the anti-sedentary services' 
	 "start of day" this will always return false.
	- For rest of the decision times (2nd to 5th), say decision time $t$, `priorAnti` is the indicator of whether there is any 
	anti-sedentary message delivered to user's phone between the decision time $(t-1)$ to the current decision time $t$. 
	- Can either be `true` or `false`

6. `lastActivity`

	- For the 2nd to 5th decision time, say decision time $t$, `lastActivity` is the indicator of whether the activity message is 
	delivered to user's phone at the previous decision time (e.g. decision time $(t-1)$)
	- For the 1st decision time, `lastActivity` is set to `false` (this is not used)
	- Can either be `true` or `false`

7. `location`
	- The classification of user's current location: `2` if currently at home, `1` if currently  at work, `0` otherwise.
	- If the current location is unknown, set to `0`.  (**Peng: confirm the latest definition of location **)
8. `watch`
	- Indicator for whether or not we have the watch-app data for the decision time. Suppose the decision time is at 12 noon. The value is `false` if the server does not have watch-app data from 11am to a randomly chosen time between 12:15 to 12:30pm. 
	- If the value is `false`, we randomize with probability 0.2 (currenlty this only occurs after MRT week, during MRT week, randomize with 0.25 whenever available). Otherwise, we use the probability calculated by the RL algorithm (after MRT week)
	

### CONDITION CHECKING

The current version of *decision* service requires the input satisfying the following conditions 
**(otherwise, the service will send Peng and Nick an email and/or write a warning with a dump of the data in a file; need to decide what to do then)**

1. [server] The *decision* service does not skip any previous decision times in the current day.

	**If this is not true, we will use default randomization probability.**
	
2. [server] All the required information is included in the input

	**If this is not true, we will use default randomization probability.**

3. [server] The type of each variable is consistent with the above description
 (e.g. `location` can only be `0`, `1` or `2`,  `availability` can only be `true` or `false` and so on.)
 
 	**If this is not true, we will use default randomization probability.**
 	
4. [server] There is no missing data in the input. 
	
	**If this is not true, we will use default randomization probability.**


## 3. Nightly Update

### WHEN TO CALL
The *nightly* service is called at every night after the 5th decision time during the study (does not include the week with no app). 
The exact time at which the *nightly* service is called needs to satisfy: 

- after we know all the required information (e.g. the app screen data and the total steps, depending on the exact definition)
- after the end of the day specified in the anti-sedentary message scheduling. That is, there cannot be any anti-sedentary message sent
 after the *nightly* service is called. 


### INPUT-OUTPUT
The *nightly* service has no output except for a message indicating successful update.  Below is an example of json input for user `1` finishing day `2`. 

~~~json
{
	"userID":1,
	"studyDay":2,
	"priorAnti":false,
	"lastActivity":false,
	"temperatureArray":[30,33.4,8.5,23.9,38.1],
	"appClick":503,
	"totalSteps":6584,
	"preStepsArray":[12,50,100,0,null],
	"postStepsArray":[300,null,100,130,31],
	"availabilityArray": [true, false, true, false, true],
	"actionArray": [false, false, false, true, false],
	"probArray": [0.5, 0, 0.1, 0, 0.2],
 	"priorAntiArray": [true, false, true, false, true],
 	"lastActivityArray": [true, false, true, false, true],
 	"locationArray": [0, 1, 2, 0, 2]
}
~~~

1. `userID`: the user ID. 

2. `studyDay`: index of the current day since the beginning of the study, starting at `1` on the first day. 

3. `priorAnti`: the indicator of whether there is any anti-sedentary message delivered to user's phone between the 5th decision time and 
the "end of the day" specified in the anti-sedentary message scheduling.  Can either be `true` or `false`. If the participant sets their 
5th decision time to a period after the anti-sedentary service's "end of day" then `priorAnti` will be set to `false`

4. `lastActivity`: the indicator of whether the activity message is delivered to user's phone at the 5th decision time in the current day.  
Can either be `true` or `false`

5. `temperatureArray`: 
  - A vector of the temperatures (in Celsius degree) at each of the five locations during the day.
  - If any of the temperatures is unknown, then the average temperature of all the participant's registered places (home and work) 
  will be substituted for the actual temperature.

6. `appClick`

  - The number of app screens encountered in the current day from 12:00 am to 11:59 pm. 
  - Mark as `null` if the heartstep server does not have the latest information about the app usage at the time of nightly updates 
  (due to no internet connection or no battery at the time)

7. `totalSteps`

  - Total of steps that are tracked by Fitbit the specific day. Fitbit defines a day as the period from 12:00am to 11:59pm in the 
  participant's local time. The data reported here will be directly collected from the 
  [Fitbit API's daily activity summary](https://dev.fitbit.com/build/reference/web-api/activity/#get-daily-activity-summary).

  - The input will be marked as `null` if any of the followings holds:

  	  (1) the participant is classfied as "not wear the sensor for the day"  based on the heart rate data (same above)
	
   	 (2) the step count can not be querired from Fitbit server (e.g. the user revokes the consent for data collection and thus 
	the Fitbit server does not have the step count data)

  	  If neither of above is true, then `totalSteps` will be a number (including 0, meaning that the pariticipant is classified
	 as "wear the sensor" and the Fibit collects 0 step during the day)

8. `preStepsArray` and `postStepsArray`

  - `preStepsArray ` is a vector of step counts collected 30 min prior to each of five decision times in the current day. 
  It is in chronological order, e.g. the first element corresponds to the 30 min pre-treatment step counts at the first decision time and so on.  

  - `postStepsArray` is for the step count 30 min after each decision time. 

  - The pre (post)- 30 min step count for a decision time will be marked as `null` if any of the followings holds:

   	 (1) the participant is classfied as "not wear the sensor during the 30 min window before (after) the decision time " 
	 based on the heart rate data (same above)

  	  (2) the step count can not be querired from Fitbit server (e.g. the user revokes the consent for data collection and thus the 
	Fitbit server does not have the step count data)

 	   If neither of above is true, then the recieved input for the correspond decision time will be a number; including 0, meaning 
	that the pariticipant is classified as "wear the sensor" and the Fibit collects 0 step during the 30 min window before (after) the decision time.

9. `availabilityArray`, `probArray`, `actionArray`, `priorAntiArray`, `lastActivityArray `, `locationArray `

	- These are the vectors of `availability`, `priorAnti `, `lastActivity ` and `location` collected at the five decision times in a day. The first element correpsonds to the first decision time and second element to the second decision time and etc. 
	- In the case where the walking suggestion service is reached by Heartsteps main server at all five decision times during the day, these information will be identical to the ones received during the day (We resend this information at the nightly update time to hanlde the case where for some reason the heatstep main server cannot contact walking suggestion service at some decision time)
	- For the decision times at which the walking suggestion service is not called by the server, the availability,  probability and action are set to FALSE, 0 and FALSE. 

### CONDITION CHECKING

The current version of *nightly* service requires the input satisfying the following conditions 
**(otherwise, the service will send Peng and Nick an email and/or write a warning with a dump of the data in a file; need to decide what to do then)**


1. [server] All the required information is included in the input

2. [server] There are five records in the input `temperatureArray`, `preStepsArray`, `postStepsArray` (including the missing ones, e.g. the ones markded as `null`)
4. [server] The type of data is consistent with the above description 
5. [server] The *nighty* service is called after the 5th decision time in the current day (e.g. the *decision* service has been called 5 times) **(Will remove this)**








