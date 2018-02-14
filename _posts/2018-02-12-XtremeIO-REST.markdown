---
layout: post
title: XtremIO REST API
date: 2018-02-12 09:42:00 -0800
---

[XtremIO](https://www.dellemc.com/en-us/storage/xtremio-all-flash.htm#collapse) is a storage array
made by DellEmc.  I can't attest to the performance of the machine because I haven't used it yet
but I can say that their REST API is terrible.  

As a large company with vast resources you would think that Dell could make a REST API correctly.
You would be wrong.  Lately I have been working on a program at work to collect performance
metrics from a variety of storage vendors.  Some of these API's are straight up crazy.  It
usually takes me no more than a minute or two to find problems.  Let me illustrate:

Run a curl command against the array with: `curl -k -s --user user:pass -X GET https://192.168.1.2/api/json/v2/types/volumes?full=1`
Here's a partial sample of the volume list output:
```
{
    "params": {
        "id-property": "vol-id"
    },
    "volumes": [
        {
            "small-io-alerts": "disabled",
            "created-by-app": "xms",
            "small-iops": "0",
            "wr-latency": "563",
            "vol-id": [
                "f6f087b30fde46e3a725238769a9c2c5",
                "test23",
                24
            ],
            "obj-severity": "information",
            "unaligned-io-alerts": "disabled",
            "unaligned-rd-bw": "0",
            "num-of-dest-snaps": 0,
            "acc-size-of-wr": "9083568951",
            "iops": "16",
            "small-io-ratio-level": "ok",
		}
	]
}
```

It's really concerning when the first curl command you run to learn the API shows terrible Json.
In this tiny piece of the volume information call you can see that numbers are being represented
as strings.  There's also fields that have numbers represented as numbers. It's not consistent.

I have tried several times in the past to get vendors to fix these problems but they seem to
not give a damn so I am going to try something new.  Lets see how they feel about me airing their dirty laundry on the internet.

A quick jump over to [json.org](http://json.org/) shows that a value in json can be
several different things.  It can be a string, number, object, etc.  Why does any of this matter?
Well for starters if someone is using this API to collect performance metrics about a cluster
and logging that to a time series database how should they interpret these numbers as strings?
When numbers are actually numbers you can code some simple logic like that goes:
```
	fn write_to_influx(input, influx_client) {
		for value in input {
			if value is String {
				influx_client.write_tag(value.name, value.value);
			}
			else if value is Number {
				influx_client.write_field(value.name, value.value);
			}
		}
	}
```
Obviously this code is just bad pseudocode but you get the idea of what is going on. You can use a simple filter to go through the values and write them to a time series database.
When numbers are strings this logic fails and you have to hand code something to examine every field and make the right decision about what to do.  This is tedious and error prone.  Another problem is that parsing this with a strongly typed programming
language is problematic.  Strongly typed languages actually care that a String is
a String and for good reason.  Representing numbers as strings requires additional logic to ensure that the numbers can be parsed correctly.  Another problem is
what size number container should I use to parse these number strings?  I'm looking
at a snapshot of one volume command here but that doesn't tell me the maximum
size I need to expect out of the fields.  To play it safe I need to use the maximum
possible number field of signed 64 bit.  This also doesn't tell me if the number
string could possibly be a floating point number at some point in its life.  The
assumptions we are making here are starting to pile up and I have only run 1 command!

This is a perfectly good reason why engineers like myself prefer open source 
solutions.  Getting someone at Dell/EMC to fix this is a nightmare.  It can take
many hours of meetings, email chains, screenshots, text files to prove what I'm saying,
convincing several layers of people I am correct and finally getting engineering to
patch it. If it was open source I could submit a pull request and argue about why the 
API should be changed. It cuts right to the chase and I engage the developers first instead of last.  

Stay tuned for the next in a series of terrible REST API's where we explore an API that
uses CSV over REST.  Yes you read that correctly.
