# [openHAB](https://www.openhab.org/) notes
Some notes on openHAB, its bindings and configuration.


__Overview:__
* [eBUS](#ebus)
* [Weather station integrations](#weather-station-integrations)
  * [Commercial hardware](#commercial-hardware)
  * [webhook/httplistener binding](#webhookhttplistener-binding)
  * [DIY (do it yourself)](#diy-do-it-yourself)

## eBUS
See [here](https://github.com/GHolli/eBUS-control) for further details on eBUS.

## Weather station integrations
### Commercial hardware
* [Fine Offset](http://www.foshk.com/) hardware/OEM manufacturer
* Wi-Fi gateway (ESP8266 based) [ecowitt GW1100](https://www.ecowitt.com/shop/goodsDetail/107#)
* [Wi-Fi Weather Stations](https://www.ecowitt.com/shop/goodsPage/1/43)

There are plenty of re-branded (ELV, Conrad, Froggit, dnt, oneConcept, Velleman, Eurochron, ...) weather stations (e.g. WH1080, WS1080, WH3080, WS3080, ...) based on this hardware.
See [here](https://wetterstationsforum.info/wiki/doku.php?id=wiki:wetterstationen) for an overview.

Some bindings for WiFi weather stations:
* [IpObserver Binding](https://www.openhab.org/addons/bindings/ipobserver/). Previously called “Ambient Weather WS-1400IP weather station” binding.
This Binding fetches the data from your local gateways web interface.
[Initial contribution](https://github.com/openhab/openhab-addons/pull/10567) to openHAB Add-ons.
* The [Ambient Weather Binding](https://www.openhab.org/addons/bindings/ambientweather/) has a quite similar name, but works very differently: it fetches the data from the ambientweather.net cloud service. No possibility to fetch the data locally.
* Ambient Weather ObserverIP Module is the name for a (hardware) gateway of Ambient Weather. Don't mix it up with the naming of the bindings.
* Retrieving the information from the web interface should be also possible via the [HTTP Binding](https://www.openhab.org/addons/bindings/http/). Not tested yet (especially the login authentication).
* A quite interesting binding, which can receive data from gateways via http GET/POST, can be found here: [webhook/httplistener binding](https://community.openhab.org/t/webhook-new-very-simple-binding-for-listening-incomming-http-requests/123597). It's not part of the official distribution yet.

### webhook/httplistener binding
Great concept, enables the gateway to actively publish the values. It's not yet included in the official bindings and has some flaws...

*Installation and setup:*

Download [here](https://github.com/ptrbojko/openhab-webhook-binding/releases/tag/initial).

Place the .jar file in the [addons](https://www.openhab.org/docs/configuration/addons.html#through-manually-provided-add-ons) folder.
<br>Then go to settings - things - "+" and add a "httplistener" thing.
<br>Once it is created, configure it in the GUI:
Settings - Things - JSON configuration - Edit script
<br>and pasting the following configuration:

```json
{
	"channels": [
		{
			"key": "body",
			"kind": "STATE",
			"state": "req.body.text"
		},
		{
			"key": "tempinf",
			"kind": "STATE",
			"state": "req.parameters.tempinf[0] + ' °F'"
		},
		{
			"key": "tempf",
			"kind": "STATE",
			"state": "req.parameters.tempf[0] + ' °F'"
		},
		{
			"key": "humidityin",
			"kind": "STATE",
			"state": "req.parameters.humidityin[0]"
		}
	]
}
```

It should be also possible by a .things file:

```
Thing httplistener:HttpListener:ecowittGateway [jsonConfig="{ \"channels\": [ { \"key\": \"tempinf\", \"kind\": \"STATE\", \"state\": \"req.parameters.tempinf[0]\" }, { \"key\": \"humidityin\", \"kind\": \"STATE\", \"state\": \"req.parameters.humidityin[0]\" } ] }"]
```
Configuration is accepted, but it does not seem to work...
So better stick with the GUI variant in this case.

**Basic example:**

_sitemap:_
```
Text item=GW_TemperatureIn
Text item=GW_Temperature
Text item=GW_Values label="Gateway values"
```
_items:_
```
Number	GW_TemperatureIn  "Temp. in[%.1f °C]"  <temperature>  {channel="httplistener:HttpListener:123456789A:tempinf"}
Number	GW_Temperature    "Temp.[%.1f °C]"     <temperature>  {channel="httplistener:HttpListener:123456789A:tempf"}
String	GW_Values                                             {channel="httplistener:HttpListener:123456789A:body"}
```
Replace "123456789A" by the actual gateway Thing ID.

__Testing the plugin without a real gateway:__

Replace "openhab" by your openHAB server IP and "thingid" by your gateway Thing ID.

*Working demo:*
```
curl --get --data 'PASSKEY=XXXXXXXXXXXXXXXXXXX&stationtype=GW1000_V1.5.4&dateutc=2021-11-02+09:54:58&tempinf=76.5&humidityin=52&baromrelin=30.345&baromabsin=29.902&tempf=66.0&humidity=72&wh26batt=0&freq=433M&model=GW1000_Pro' http://openhab:8080/httplistener/thingid/any-further-url-path
```
This calls mimic my hardware gateway:

_ecowitt GW1100, Ecowitt protocol_ (throws exception, fails):
```
curl --data 'PASSKEY=0123456789ABCDEF0123456789ABCDEF&stationtype=GW1100A_V2.0.9&dateutc=2022-03-10+20:01:02&tempinf=66.0&humidityin=43&baromrelin=28.762&baromabsin=28.762&tempf=48.0&humidity=42&winddir=163&windspeedmph=1.57&windgustmph=2.24&maxdailygust=6.93&solarradiation=12.48&uv=0&rainratein=0.000&eventrainin=0.000&hourlyrainin=0.000&dailyrainin=0.000&weeklyrainin=0.000&monthlyrainin=0.000&yearlyrainin=1.079&totalrainin=1.079&temp1f=66.7&humidity1=41&temp2f=48.0&humidity2=38&temp3f=60.6&humidity3=52&temp4f=51.8&humidity4=57&temp5f=49.8&humidity5=55&temp6f=44.1&humidity6=47&temp7f=69.4&humidity7=39&temp8f=59.4&humidity8=42&wh65batt=0&batt1=0&batt2=0&batt3=0&batt4=0&batt5=0&batt6=0&batt7=1&batt8=0&freq=868M&model=GW1100A' http://openhab:8080/httplistener/thingid/any-further-url-path
```
_ecowitt GW1100, Wunderground protocol_ (throws exception, fails):
```
curl --get --data 'ID=StationID&PASSWORD=StationKey&tempf=47.7&humidity=42&dewptf=25.7&windchillf=47.7&winddir=202&windspeedmph=1.34&windgustmph=3.36&rainin=0.000&dailyrainin=0.000&weeklyrainin=0.000&monthlyrainin=0.000&yearlyrainin=1.079&solarradiation=12.48&UV=0&indoortempf=65.8&indoorhumidity=43&baromin=28.753&lowbatt=0&dateutc=now&softwaretype=GW1100A_V2.0.9&action=updateraw&realtime=1&rtfreq=5' http://openhab:8080/httplistener/thingid/any-further-url-path
```

### DIY (do it yourself)
Self-made RF to WiFi MQTT gateway:
<!-- ![RF to WiFi MQTT gateway](doc/images/IMG_2474x.JPG?raw=true) ![RF to WiFi MQTT gateway](doc/images/IMG_2477x.JPG?raw=true) -->
<img src="doc/images/IMG_2474x.JPG?raw=true" alt="RF to WiFi MQTT gateway" width="250"><img src="doc/images/IMG_2477x.JPG?raw=true" alt="RF to WiFi MQTT gateway" width="250">
<p>Component list:

* Some ESP8266 module
* [HopeRF RFM69CW](https://www.hoperf.com/modules/rf_transceiver/RFM69C.html) or [TI CC1101](https://www.ti.com/product/CC1101)
* [Optional] Microcontroller board (e.g. STM32F103C8T6)

I used the additional microcontroller for decoding the RF signal and sending it to the ESP8266 via serial interface.
