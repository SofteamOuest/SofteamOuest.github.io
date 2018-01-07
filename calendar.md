---
layout: page
title: Calendrier
permalink: /calendar/
---

<script src="https://code.jquery.com/jquery-3.1.1.min.js"   
integrity="sha256-hVVnYaiADRTO2PzUGmuLJr8BLUSjGIZsDYGmIJLv2b8="  crossorigin="anonymous"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.20.1/moment.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/fullcalendar/3.2.0/fullcalendar.min.js"></script>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/fullcalendar/3.2.0/fullcalendar.min.css">
<link rel="stylesheet" media="print" href="https://cdnjs.cloudflare.com/ajax/libs/fullcalendar/3.2.0/fullcalendar.print.css">

<script>
$(document).ready(function() {

	$('#calendar').fullCalendar({
		events:'https://softeamouest.github.io/calendar-data'
	})

});

</script>

<!--
{% for event in site.events %}
{{event.title}} {{event.event_date}}<br/>
{% endfor %}
-->
<div id="calendar"></div>
