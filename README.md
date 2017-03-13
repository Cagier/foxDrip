foxDrip - An Introduction
=========================

A foxDrip is a device that requires no bluetooth, no wifi and no mobile phone signal and allows the user to relay blood glucose readings remotely from a G4 CGM sensor without the need to carry a mobile phone.  (It still sends these signals to an uploader phone or android tablet but that device does not need to be even in the same country and so it can theoretically be left at home, connected over WiFi.)  So how does it work?

Sigfox is a global Low-Power Wide-Area network (LPWAN).  It is commonly used with "Internet of Things" devices as an effective way of getting data onto the internet.  It is an extremely cost-efficient solution as the annual Sigfox subscription fee is about €15 per year.  It is completely separate from traditional phone networks and so no SIM or any subscription is required.  More information can be found at https://www.sigfox.com and you can check your local coverage on https://www.sigfox.com/coverage

xDrip is an established solution for uploading data into Nightscout servers.  By using an xDrip bridging device (or the now more commonly used xBridge2 device) data from a Dexcom G4 CGM system can be retrieved and sent via a bluetooth signal to an uploader phone which then processed the signal and calculated blood sugar readings for upload to the internet.  In recent times, the xDrip app on the phone has been enhanced to retrieve information from a number of other sources allowing a variety of devices to send information over wifi and 2G.  More information can be found on http://stephenblackwasalreadytaken.github.io/xDrip/

By combining these two technologies, we can eliminate the restriction of the short-range bluetooth connection and build a compact device (that fits in a tic tac box) which is capable of uploading data directly into the Nightscout infrastructure without being in the proximity of the uploader phone and requires no wifi or cellphone signal.  Because this merges the Sigfox and xDrip technologies it is called a "foxDrip".

Sigfox network coverage will vary from place to place and it will naturally work better outside.  Also, the use of the miniaturised antenna may also reduce performance.  On that basis, this solution may not completely eliminate the requirement for a traditional xDrip (depending on your location) and may not be suitable for everyone but at the very least it is an excellent supplemental device in situations where you may not be able to carry an uploader phone with you - for example, when playing sport or in areas with poor mobile phone coverage.

It may also be a good alternative to a "parakeet" or "mDrip" as it may work out cheaper to run or work better in certain areas that have poor cell phone coverage.  It is also a good potential alternative to a "NodeMCU" or "yDrip" if you have no access to WiFi or want a more portable solution (and it might be worth considering using the bigger 6 inch antenna in this scenario).

This device is very straightforward to build and set up.  It requires no additional servers (such as google VMs or Dexie servers) to be created, although there is some very simple and quick backend set up in the Sigfox back-end to get the data to securely send to Nightscout.

Detailed instructions are provided to get everything set up quickly.



How to make a foxDrip system
============================

1) Build the foxDrip Hardware device.

a) Partslist:
This is similar to the xDrip hardware but we use a sigfox module instead of a bluetooth module:
	Wixel
	LiPo battery
	USB Lipo Charger
	A tic-tac box
	Sigfox Breakout board BRKWS01 - available from https://yadom.fr/carte-breakout-sfm10r1.html
	Any suitable miniature uFL antenna - I used this one here https://ie.rs-online.com/web/p/products/7043417/

Any 850-900Mhz uFL antenna should work in Europe.  868Mhz is the actual Sigfox frequency range in Europe.

The sigfox kit above comes with a larger antenna (which is good for initial testing) and 1 year's free subscription to the Sigfox network. Other boards and modules will probably work also but this one is nice and small and uses an extremely simple AT command set.  Also, the board above is a European version but there are other similar modules that work in the US and ASia which should be reasonably interchangeable.  However, if using another board then some tweaks to the foxdrip.c source might be required so contact me for advice on this (preferably before purchasing anything) if you are considering using an alternative module.


b) Load Software:
Compile and load the foxdrip software onto the Wixel (in same way as the you would the dexdrip.wxl or xbridge2.wxl files).  Remember to change your transmittter ID within the foxdrip.c code before compiling if you do not want to receive signals from other nearby G4 CGM users.  If that doesn't bother you then you don't need to worry about the rest of the repository or the wixel development bundle and you can just download the prepared foxdrip.wxl file for transfer to the Wixel with the standard pololu Wixel Configuration Utility.  Detailed instructions on how to load the foxdrip.wxl app onto the wixel can be found at https://www.pololu.com/docs/0J46/3.d


c) Construction:
Wire it all together as per the circuit diagram, soldering the connections one by one before connecting the antenna and the battery.
(I'll elaborate on this a bit later but if you have built an xDrip before then it is nearly the same.)
IMPORTANT: Make sure you have an antenna properly connected to the Sigfox board before powering up the foxDrip or you could damage the transmitter unit.  So please connect the battery as the last step!  (Also, be aware that once you have everything wired then connecting a USB cable to the Wixel will power the sigfox board as well.)



2) Set up the Sigfox Backend
There are a couple of steps to add your device to the network and get it to communicate with Nightscout.

a) Set up the Sigfox subscription
As per the instructions that come with the Sigfox board, this is very quick and straightforward.
The exact details may vary, depending on what country/region that you are in.  For the board I used, I just had to visit http://snoc.fr/sigfoxactivate which directed me to my local country's network provider.  (In Ireland we use VT Networks who piggy-back on the Saorview transmission sites, amongst other locations.)  There you simply enter your Sigfox PAC ID and PAC to activate the device subscription.

b) Set up the Callback parameters
Log in to  https://backend.sigfox.com
* Go to Device Type and click on the Name of your device
* Click on "CALLBACKS" on the bottom left
* Click on "New" near the top on the right-hand side
* Select "Custom callback" and use the parameters below:
	Type: Data/ Uplink
	Channel: URL
	Custom Payload Configuration: TransNo::uint:8 TxID::char:5 Raw::uint:20 Filtered:8:uint:20::3 Bat::uint:8
	URL Pattern: https://api.mlab.com/api/1/databases/your_dbname/collections/Sigfox?apiKey=your_api_key
	Use HTTP Method: POST
	Send SNI: Does not need to be ticked
	Content type: application/json
	Body:
{
 "SigfoxData" : "{data}",
 "TransmissionId": {customData#TransNo},
 "TransmitterId": "{customData#TxID}",
 "RawValue": {customData#Raw},
 "FilteredValue": {customData#Filtered},
 "BatteryLife": {customData#Bat},
 "CaptureDateTime": {time}000,
 "ReceivedSignalStrength": -99,
 "UploaderBatteryLife": 100,
 "SigfoxDevice": "{device}",
 "SigfoxSnr":{snr},
 "SigfoxStation": "{station}",
 "SigfoxAvgSnr" : "{avgSnr}",
 "SigfoxRssi" : {rssi},
 "SigfoxSeqNo" : {seqNumber}
}


You should be able to copy and paste the payload, URL and Body information from here and paste it into that screen.
The ONLY thing that you will need to change is the "URL PATTERN" to match your own database.  So where the above text says:

	https://api.mlab.com/api/1/databases/your_dbname/collections/Sigfox?apiKey=your_api_key

you need to change two things:
* Replace "your_dbname" with the name of the mlab database that you use for Nightscout - e.g. "mycgmdb".
* Replace "your_api_key" with the api key generate by mlab - e.g. "2E81PUmPFI84t7UIc_5YdldAp1ruUPKye"

So an example URL pattern would be something like
	https://api.mlab.com/api/1/databases/mycgmdb/collections/Sigfox?apiKey=2E81PUmPFI84t7UIc_5YdldAp1ruUPKye


The details of how to set up the api key can be found at http://docs.mlab.com/data-api/
and are as follows:
	> Log in to the mLab management portal
	> Click your username (not the account name) in the upper right-hand corner to open your account user profile
	> If you are already in the account details page, then click on the row with your username in the Account Users section
	> If the status is showing as “Data API Access: Disabled” in the “API Key” section, click the “Enable Data API access” button
	> Once Data API access is enabled, your current API key will be displayed in the “API key” field 



3) Set up the xDrip app to receive the data
Install the xDrip application on your uploader phone, if you have not already done so.  First, set it up in the normal way that you would set up a standard xDrip/xbridge.  Then, in the "Settings", change the following two fields:

>>  Hardware Data Source:
	Wifi Wixel (foxDrip only) OR:
	Wifi Wixel + xBridge Wixel (both foxDrip and standard xDrip)

>>  List of receivers:
If you already use this with a yDrip, mDrip, NodeMCU, parakeet, etc. then you will already be familiar with this step.  Basically you add in a list of devices and databases where your data is available.  For foxDrip, you want to add your new mlab "Sigfox" collection.  The required entry should look something like
	mongodb://Keith:myDBpassword@ds053611.mongolab.com@53611/mycgmdb/Sigfox


The example given in the help for this box is normally mongodb://user:pass@ds053598.mongolab.com@53598/db/collection.
As "Sigfox" is the collection name used in the Sigfox backend above then we will use that here so the format is
	mongodb://user:pass@ds053598.mongolab.com@53598/db/Sigfox
You will obviously need to modify this further to use your own mlab details.  If you use the "MongoDB" option to upload your data to Nightscout then it will be nearly identical to the URI that is in that parameter, except you add "/Sigfox" to the end (with no quotes).

If you are still having trouble working out what this should look like for you, log into mlab and select your database.  There will be some text in the top left that should help you with these parameters.  It will say:
	"To connect using a driver via the standard MongoDB URI (what's this?):"
	mongodb://<dbuser>:<dbpassword>@ds053611.mlab.com:53611/mycgmdb
So, if you just modify this to read
	mongodb://<dbuser>:<dbpassword>@ds053611.mlab.com:53611/mycgmdb/Sigfox
and replace <dbuser> and <dbpassword> with your credentials then you will get something that looks like
	mongodb://Keith:myuserpassword@ds053611.mongolab.com@53611/mycgmdb/Sigfox

Please note that it is very important NOT to use the User name and password that you use to log into mlab.  You must instead use the the DATABASE User Name and Password.  If you need a reminder, the "Users" tab is between the "Collections" and "Stats" tabs in mlab and the valid database users should be listed there.  However, the password is not shown there so hopefully you can remember that yourself or have it written down somewhere!  If all else fails you can add another User on that screen and use that instead...



4) Restrictions
Devices on the Sigfox network are theoretically limited to 140 per day (which comes from the legal ETSI 1% rule).  As the G4 transmits approximately double these amount of readings in a day (288 if no missed readings), the options are to programatically restrict the foxdrip to only send a reading every 10 minutes (instead of every 5 minutes) or to ignore this and just to only use the foxdrip for up to 12 hours a day.  I have gone with the latter option for the moment as you are technically exempt from this limit if developing prototypes.  As foxDrips are not commercial products and based on development boards then all such units fall into this category as far as I can determine.  However, it is something that I might look at in more detail in future.




Troubleshooting:
================

Hopefully it will all just magically work but if you are struggling to get it going then here are some things to check out...

* Firstly, re-check your soldering and wiring to make sure everything is hooked up.
* Plug in a USB cable to the LiPo charger to ensure there is power getting to the rig and the battery is charged.
* If the yellow light does not start flashing when power is initially connected then either there is a wiring problem or the software has not yet been loaded on the wixel correctly.
* If testing indoors, try using a bigger antenna first and check if positioning the unit closer to a window makes a difference.
* For the technically inclined, if you want to add in a few LEDs (even temporarily) to the sigfox board then you can easily observe CPU, UART and Radio activity to confirm that something is actually happening.  You could even risk just holding one in place if you are feeling adventurous!
* For the technically inclined, if you happen to have and FTDI (TTL) cable you can use a console program to look at the data coming from the Wixel to make sure that the commands are being sent to sigfox board (on the sigfox RX pin).  You can also check the sigfox TX pin as the board should send out an "OK" after each AT command is successfully executed.  You should just need to connect or hold the cable's GND and RX pins to the board to see this data.  (There is no need to use the red power or TX wires on the cable.)
* Ensure data is getting to the Sigfox network by looking at the statistics.  Pay attention to the Rssi and SNR to ensure that you are getting a strong clean signal out to the Sigfox network.
* On the Sigfox backend, it can also be useful to add another Custom callback and use an "EMAIL" channel instead of a "URL" channel.  This will direct the Sigfox messages to an email address of your choosing so you can inspect the contents there.
* Check that data is getting to the mlab database.  If you log in to mlab, there should be a "Sigfox" collection automatically created once you start getting data from Sigfox and a new entry should appear in there for each BG reading.
* If you are getting mlab data but no entries are appearing in the xDrip app then chances are that you have not got the "List of receivers" set up correctly (wrong password, missing colon or something like that).
* If you are getting mlab data but no entries are still not appearing in the xDrip app then go into Settings and "View Recent Errors/Warnings".  That may give some indication of what your issue is but still the chances are that the cdrip settings above are wrong.
* If you are still stuck then give me a shout over Github or raise an issue.



And finally...
==============

Like the mDrip and the yDrip, this is a labour of love.  I'd really appreciate any feedback that you have on this project.  Please share your experiences and message me via Github with any queries or compliments!  ;)
