# TheLostCalendar

Let me tell you the story of Google Admin and the raiders of the lost Calendar !
It all began when an adventurer deleted a team calendar instead of unsubscribe from it in Google Calendar.

![Alt text](/1.png?raw=true "All was lost")

All was lost for the fellowship and its members because the calendar cannot be restore once vanished.
And the adventurer memory was not enough to recreate all the events…

Hope was lost but with thorough research and some digging, we found some lead : restore the events instead of the calendar !

First you have get the deleted calendar ID in Google Workspace Calendar Logs. 
To find it, filter the search with the user who deleted the calendar and the event : Calendar Deleted.

![Alt text](/2.png?raw=true "Admin logs 1")

Then search in Google Calendar Logs all events deleted around calendar deletion time. 

![Alt text](/3.png?raw=true "Admin logs 2")

Because the calendar deletion leads to deletion of almost all events in the calendar, this search will get you events ids to restore. 
Once the search is done, you can export it in a Google Sheets.

Our adventure does not stop here ! 
Unfortunately, the previous step does not give us the start/end for the deleted events.

But the mighty Google Workspace BigQuery logs can ! 
You can run the following query to get results in Google Sheets.
```SQL
SELECT calendar.event_id,
string_agg(calendar.event_guest) as guest,
string_agg(CAST (calendar.end_time AS STRING)) as startTime,
string_agg(CAST (calendar.end_time AS STRING)) as endTime,
string_agg( DISTINCT calendar.event_title) as title
FROM `dashboard.gsuite_logs_dataset.activity`
WHERE 
TIMESTAMP_TRUNC(_PARTITIONTIME, DAY) > TIMESTAMP("beginning of time")
and event_name in("add_event_guest","create_event")
and calendar.calendar_id=deleted_calendar_id
group by 1
```

This export will have all the events created in the deleted calendar with their guests and times. 

Afterwards, you shall link the two exports : 
- The one from the Admin Console (deleted events) 
- The one from Big Query (events created in calendar) 

The link will be made by the event id (with some magic Google Sheets formulas)

![Alt text](/4.png?raw=true "Sheets")

Once the link is done, the event in CONSOLE export (the ones to recreate) will have their informations : title, guests, start/end time !
Finally you can create a new team calendar and create the lost events in it by running a simple but powerful script from the Google Sheets 
```javascript
function createEvents() {
  let cal = CalendarApp.getCalendarById(your_calendar_id)
  let sp = SpreadsheetApp.openById(your_google_sheet_id)
  let sh = sp.getSheetByName("CONSOLE")
  //Filter on event having an start and end date
  let data = sh.getDataRange().getValues()
    .filter(x => x[2] != "").filter(x => x[3] != "").slice(1)
  //Loop over the data to create the events
  data.forEach(function (t) {
    cal.createEvent(t[1], new Date(t[2]), new Date(t[3]), { guests: t[4]})
  })
}

```


Our story ends here with some things to consider : 
- Some event might not be recovered : recurring event, events without logs,...
- Be aware, and make your colleagues are, to not delete calendar unless sure of it
- Google Workspace Big Query logs are often a good place to dig and find your treasure  



