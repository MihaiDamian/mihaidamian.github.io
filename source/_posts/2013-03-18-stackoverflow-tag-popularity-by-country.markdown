---
layout: post
status: publish
published: true
title: StackOverflow tag popularity by country
categories:
- data
tags:
- StackExchange
- StackOverflow
- Google Chart Tools
comments: true
---
StackExchange provides a great little tool called <a href="http://data.stackexchange.com" title="Data Explorer" target="_blank">Data Explorer</a> that lets any user run readonly SQL queries on their Q&A sites. I decided to put it to the <a href="http://data.stackexchange.com/stackoverflow/query/101591/originating-posting-locations-for-a-tag" title="test" target="_blank">test</a> and use it to find how many questions got posted from each country for a set of tags. The location information of each question author is declared voluntarily in freeform format. This means the data collected is pretty noisy, if available at all. To map the raw location to a canonical country name I used MapQuest's open <a href="http://open.mapquestapi.com/geocoding/" title="geocoder" target="_blank">geocoder</a> - one of few such services not placing  restrictions on the number of daily requests. Below is the resulting map showing the breakdown of questions by country and tag:

<script src="//ajax.googleapis.com/ajax/libs/jquery/1.11.0/jquery.min.js"></script>
<script type='text/javascript'>
    jQuery(document).ready(function() {
        var map = jQuery('#map-frame');
		var newWidth = jQuery('#map-wrapper').width();
        var minDropdownMenuHeight = 455;
        var newHeight = Math.max(newWidth * (377.0 / 556), minDropdownMenuHeight);
		map.width(newWidth).height(newHeight);
    });
</script>
<div id="map-wrapper">
	<iframe id="map-frame" src="/assets/2013-03-18-stackoverflow-tag-popularity-by-country/iFrame/index.html" style="border:0; width:100%; height:500px;" seamless></iframe>
</div>
