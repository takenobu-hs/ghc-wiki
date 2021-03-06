# The Trac Roadmap






The roadmap provides a view on the [ticket system](trac-tickets) that helps planning and managing the future development of a project.


## The Roadmap View



A roadmap is a list of future milestones. The roadmap can be filtered to show or hide *completed milestones* and *milestones with no due date*. In the case that both *show completed milestones* and *hide milestones with no due date* are selected, *completed* milestones with no due date will be shown.


## The Milestone View



A milestone is a future timeframe in which tickets are expected to be solved. You can add a description to milestones (using [WikiFormatting](wiki-formatting)) describing main objectives, for example. In addition, tickets targeted for a milestone are aggregated, and the ratio between active and resolved tickets is displayed as a milestone progress bar. It is possible to further [
customise the ticket grouping](http://trac.edgewall.org/intertrac/TracRoadmapCustomGroups) and have multiple ticket statuses shown on the progress bar.



It is possible to drill down into this simple statistic by viewing the individual milestone pages. By default, the active/resolved ratio will be grouped and displayed by component. You can also regroup the status by other criteria, such as ticket owner or severity. Ticket numbers are linked to [custom queries](trac-query) listing corresponding tickets.


## Roadmap Administration



With appropriate permissions it is possible to add, modify and remove milestones using either the web interface (roadmap and milestone pages), web administration interface or by using `trac-admin`. 



**Note:** Milestone descriptions can not currently be edited using `trac-admin`.


## iCalendar Support



The Roadmap supports the [
iCalendar](http://www.ietf.org/rfc/rfc2445.txt) format to keep track of planned milestones and related tickets from your favorite calendar software. Many calendar applications support the iCalendar specification including:


- [ Apple iCal](http://www.apple.com/ical/) for Mac OS X.
- [
  Mozilla Calendar](http://www.mozilla.org/projects/calendar/), cross-platform.
- [
  Korganizer](http://kontact.kde.org/korganizer/), the calendar application of the [
  KDE](http://www.kde.org/) project.
- [
  Evolution](https://wiki.gnome.org/Apps/Evolution), a contact manager, address manager and calendar for Gnome.
- [
  Microsoft Outlook](http://office.microsoft.com/en-us/outlook/) can also read iCalendar files and appears as a new static calendar in Outlook.
- [ Google Calendar](https://www.google.com/calendar/).
- [
  Chandler](http://chandlerproject.org), a personal and small-group task management and calendaring tool, Apache licensed and orphaned since 2009.


To subscribe to the roadmap, copy the iCalendar link from the roadmap (found at the bottom of the page) and choose the "Subscribe to remote calendar" action (or similar) of your calendar application, and insert the URL just copied.



**Note:** For tickets to be included in the calendar as tasks, you need to be logged in when copying the link. You will only see tickets assigned to yourself and associated with a milestone.



**Note:** To include the milestones in Google Calendar you might need to rewrite the URL:


```
RewriteEngine on
RewriteRule ([^/.]+)/roadmap/([^/.]+)/ics /$1/roadmap?user=$2&format=ics
```


More information about iCalendar can be found at [
Wikipedia](http://en.wikipedia.org/wiki/ICalendar).


---



See also: [TracTickets](trac-tickets), [TracReports](trac-reports), [TracQuery](trac-query), [
TracRoadmapCustomGroups](http://trac.edgewall.org/intertrac/TracRoadmapCustomGroups)


