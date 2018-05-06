# Investigating a Drop in User Engagement

# The problem

> You show up to work Tuesday morning, September 2, 2014. The head of the Product team walks over to your desk and asks you what you think about the latest activity on the user engagement dashboards. You fire them up, and something immediately jumps out:

https://modeanalytics.com/modeanalytics/reports/cbb8c291ee96/runs/7925c979521e/viz1/cfcdb6b78885

> The above chart shows the number of engaged users each week. Yammer defines engagement as having made some type of server call by interacting with the product (shown in the data as events of type “engagement”). Any point in this chart can be interpreted as “the number of users who logged at least one engagement event during the week starting on that date.”

> You are responsible for determining what caused the dip at the end of the chart shown above and, if appropriate, recommending solutions for the problem.

# Getting oriented

> Before you even touch the data, come up with a list of possible causes for the dip in retention shown in the chart above. Make a list and determine the order in which you will check them. Make sure to note how you will test each hypothesis. Think carefully about the criteria you use to order them and write down the criteria as well.

**Potential Causes:**
1. *Artificially inflated numbers*: Maybe the increase over the previous few weeks is the real error, and the drop is simply a correction to this and reflects the true engagement. I would look into things like traffic from automated bots or other connections that don't really signify a user actively engaging with the service.
2. *Error preventing users from connecting*: Perhaps there is some error preventing users from accessing the service. I would check the types of connections users were making, and see if there was suddenly a drop in a specific type of connection that would indicate an error.
3. *Real world event*: There might be some sort of real world event that caused either several users to sign up in the preceding weeks or several users to quit around the time of the decrease. I would check user forums to see if there was any major discussion going on. 
4. *User purge*: Maybe several users were not actively engaged in the service, their only connections coming from some sort of automated API call and their accounts were removed which stopped that traffic. I would check to see how many users were deactivated around the time of the decrease in engagement.

# Digging in

> Once you have an ordered list of possible problems, it’s time to investigate.

> For this problem, you will need to use four tables. The tables names and column definitions are listed below—click a table name to view information about that table. 

## Table 1: Users (`tutorial.yammer_users`)

> This table includes one row per user, with descriptive information about that user's account.

- **`user_id`**:	A unique ID per user. Can be joined to user_id in either of the other tables.
- **`created_at`**:	The time the user was created (first signed up)
- **`state`**:	The state of the user (active or pending)
- **`activated_at`**:	The time the user was activated, if they are active
- **`company_id`**:	The ID of the user's company
- **`language`**:	The chosen language of the user

## Table 2: Events (`tutorial.yammer_events`)

> This table includes one row per event, where an event is an action that a user has taken on Yammer. These events include login events, messaging events, search events, events logged as users progress through a signup funnel, events around received emails.

- **`user_id`**:	The ID of the user logging the event. Can be joined to user\_id in either of the other tables.
- **`occurred_at`**:	The time the event occurred.
- **`event_type`**:	The general event type. There are two values in this dataset: "signup_flow", which refers to anything occuring during the process of a user's authentication, and "engagement", which refers to general product usage after the user has signed up for the first time.
- **`event_name`**:	The specific action the user took. Possible values include:
    - *`create_user`*: User is added to Yammer's database during signup process
    - *`enter_email`*: User begins the signup process by entering her email address
    - *`enter_info`*: User enters her name and personal information during signup process
    - *`complete_signup`*: User completes the entire signup/authentication process
    - *`home_page`*: User loads the home page
    - *`like_message`*: User likes another user's message
    - *`login`*: User logs into Yammer
    - *`search_autocomplete`*: User selects a search result from the autocomplete list
    - *`search_run`*: User runs a search query and is taken to the search results page
    - *`search_click_result_X`*: User clicks search result X on the results page, where X is a number from 1 through 10.
    - *`send_message`*: User posts a message
    - *`view_inbox`*: User views messages in her inbox
- **`location`**:	The country from which the event was logged (collected through IP address).
- **`device`**:	The type of device used to log the event.

## Table 3: Email Events (`tutorial.yammer_emails`)

> This table contains events specific to the sending of emails. It is similar in structure to the events table above.

- **`user_id`**:	The ID of the user to whom the event relates. Can be joined to user_id in either of the other tables.
- **`occurred_at`**:	The time the event occurred.
- **`action`**:	The name of the event that occurred. "sent_weekly_digest" means that the user was delivered a digest email showing relevant conversations from the previous day. "email_open" means that the user opened the email. "email_clickthrough" means that the user clicked a link in the email.

## Table 4: Rollup Periods (`benn.dimension_rollup_periods`)

> The final table is a lookup table that is used to create rolling time periods. Though you could use the INTERVAL() function, creating rolling time periods is often easiest with a table like this. You won't necessarily need to use this table in queries that you write, but the column descriptions are provided here so that you can understand the query that creates the chart shown above.

- **`period_id`**:	This identifies the type of rollup period. The above dashboard uses period 1007, which is rolling 7-day periods.
- **`time_id`**:	This is the identifier for any given data point — it's what you would put on a chart axis. If time_id is 2014-08-01, that means that is represents the rolling 7-day period leading up to 2014-08-01.
- **`pst_start`**:	The start time of the period in PST. For 2014-08-01, you'll notice that this is 2014-07-25 — one week prior. Use this to join events to the table.
- **`pst_end`**:	The start time of the period in PST. For 2014-08-01, the end time is 2014-08-01. You can see how this is used in conjunction with pst_start to join events to this table in the query that produces the above chart.
- **`utc_start`**:	The same as pst_start, but in UTC time.
- **`pst_start`**:	The same as pst_end, but in UTC time.

# Making a recommendation

> Start to work your way through your list of hypotheses in order to determine the source of the drop in engagement. As you explore, make sure to save your work. It may be helpful to start with the code that produces the above query, which you can find by clicking the link in the footer of the chart and navigating to the “query” tab.

```sql
SELECT DATE_TRUNC('week', e.occurred_at),
       COUNT(DISTINCT e.user_id) AS weekly_active_users
  FROM tutorial.yammer_events e
 WHERE e.event_type = 'engagement'
   AND e.event_name = 'login'
 GROUP BY 1
 ORDER BY 1
```

## Testing my hypotheses

### 1. Artificially inflated numbers

I will look at table 2, specifically "engagement" type events and specifically look to see if there is anything interesting in the `device` column that might indicate how involved the user actually was.

```sql
SELECT device
FROM tutorial.yammer_events 
GROUP BY 1
```

That query shows me 26 unique devices that users have utilized to connect to the service, and they all appear to be user devices (ie not a server) so I don't think I'll be able to test this theory more.

### 2. Error preventing users from connecting

For this I will again look at table 2, but expand to look at signup events as well as engagement. Maybe there are fewer user signups due to some error in that process. I'm not sure how specific I'll be able to get given these tables, but I may get some leads.

```sql
SELECT
  DATE_TRUNC('week', created_at) AS week,
  COUNT(*) AS users
FROM tutorial.yammer_users
GROUP BY 1
ORDER BY 1
```

This shows that there *was* a drop in signups the week of the decrease, but signups continued to rise after a week so if there was an error it seems to have been corrected.

### 3. Real world event

This might be hard to determine given the tables I have access to, but perhaps a significant increase in weekly_digest emails from table 3 would indicate that there is a lot of conversation going on which could lead to further investigations.

```sql
SELECT
  DATE_TRUNC('week', occurred_at) AS week,
  COUNT(user_id) AS weekly_digests
FROM tutorial.yammer_emails
WHERE action = 'sent_weekly_digest' 
GROUP BY 1
ORDER BY 1
```

This shows that weekly digest emails have steadily risen over the last several weeks, but there is no spike or decrease around the time in question so it doesn't appear that there are any conversations going on that are particularly active.

### 4. User purge

I don't know that the users table will show user deactivations, but I could potentially look at the date of a users most recent engagement.

```sql
SELECT
  week,
  COUNT(user_id) AS user_count
FROM (
  SELECT
    user_id,
    MAX(DATE_TRUNC('week', occurred_at)) AS week
  FROM tutorial.yammer_events 
  WHERE event_type = 'engagement'
  GROUP BY 1
) AS q
GROUP BY 1
ORDER BY 1
```

It does not appear that the number of users who have not engaged with the platform for several weeks was dramatically large, so it is unlikely that a significant number of users were dropped.

# Answer the following questions:

> Do the answers to any of your original hypotheses lead you to further questions?

There was certainly a drop in user signups during the week of the decrease in total engagement, so I would like to investigate that further.

> If so, what are they and how will you test them?

I would want to look further into the details of the drop in user signups, such as the referral source of users arriving at Yammer.

> If they are questions that you can’t answer using data alone, how would you go about answering them (hypothetically, assuming you actually worked at this company)?

I would ask around to see if there were any special incentives for signups that may have expired around the time of the decrease.

> What seems like the most likely cause of the engagement dip?

It looks like there was a drop in user signups, but I don't know that it was enough to cause all the decrease in the total engagement.

> What, if anything, should the company do in response?

Investigate further this issue of user signups.