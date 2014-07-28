---
comments: true
date: 2014-07-21 13:38:11
layout: post
slug: developing-your-first-galaxy-gear-app
title: Developing your first Tizen Galaxy Gear2 App - Tips
summary: Samsung Galaxy Gear2 smartwatches are some nice new wearables from Samsung. Although we already know that Android Wear is on its way, it will take some time for it to arrive. Why not take this opportunity to develop your first app for Gear2?
author: Adam Nadoba
tags:
- Galaxy Gear2
- Tizen SDK
- Samsung
- Wearables
- Android
---

Samsung Galaxy Gear2 smartwatches are some nice new wearables from Samsung. As you probably know, creating a widget for Gear2 involves some HTML, CSS and last, but not least, JavaScript skills. In other words - we will be developing on Tizen OS.

Gear2 hit the market only 3 months ago so there aren't many noteworthy tutorials on the web. Unfortunately, only [2 decent official video tutorials](http://developer.samsung.com/samsung-gear#) are available. Besides that you can find some valuable information on Gear by looking up the [denvycom.com blog](http://denvycom.com/blog/?s=gear+2) and... To be honest - that's all we've got.

Our WearLabs™ team recently managed to develop an [Android application, which successfully takes advantage of Galaxy Gear2 functionalities](http://fieldsofwar.co/). I was the one to focus on the smartwatch side of the game and actually really enjoyed it. That's why I came up with an idea to share what I've learned and maybe encourage a few of you to develop some new fancy Gear2 apps.

Code shown in this post can be [downloaded from my repository](https://github.com/ScalaConsultants/galaxy-gear2-tutorial). Feel free to play with it or even use it in your own projects.

##Setting up the environment & troubleshooting
Follow the [official install guide](http://developer.samsung.com/samsung-gear#) carefully and you're all OK. Remember about dealing with certification matters, because you will NOT be able to run your own applications, nor will you be able to build and launch existing tutorial ones. Don't forget to push the certificate into Galaxy Gear device. Also be sure to enable USB debugging on smartwatch and check 'Unknown sources' option in the Gear Manager app.

##Understanding the application
Now, when your IDE is ready, it's time to install and import [official HelloAccessory tutorial project](http://img-developer.samsung.com/contents/cmm/HelloAccessory.zip). As you can see the app is divided into 2 separate pieces: the Android host-side, and the wearable/gear-side. This is the most common scenario called integrated app type. Android-side service-based class runs the logic, which controls the Gear widget.

~~~ android
public class HelloAccessoryProviderService extends SAAgent {

	// ...

	public HelloAccessoryProviderService() {
			super(TAG, HelloAccessoryProviderConnection.class);
		}
		
		public class HelloAccessoryProviderConnection extends SASocket {
		
			// ...
~~~

Here's how the skeleton of host-side Provider Service looks like. SAAgent class extends the standard Android Service class. In other words, this file represents a special Service, which contains all host-side functionalities. There's also SASocket subclass, which drives the connection itself, between Android smartphone and Gear2 widget. It's your application's main logic.

In general - SAAgent class can be compared to something like a Peer. On the other hand, SASocket is the 'bridge' between SAAgent and Galaxy Gear2 device.

##Gear-side logic
Information from the previous paragraph comes very handy when we take a closer look at the Gear-side part of the application. Open up the JavaScript file and compare it with the following method flow diagram:
![wtf](http://yuml.me/b314b50f)

When Gear tries to connect, it searches for available SAAgent. Then, we go one step further by setting appropriate listener and firing 'SAAgent.findPeerAgents' method. Next, we check, if SAAgent's and Gear's AppNames correspond with each other. If so, we request service connection with SAAgent and finally can communicate with the host-side Android device using SASocket.

##Sending and receiving data
Here comes the most important part of the first tutorial app - transferring data between Android and Gear. We do that using binary data arrays (in this example, Strings). It's a very crucial skill, essential basic, which you must know in order to be able to jump into developing bigger and more valuable applications. Let's take a closer look at the Android-side:

###Receiving - Android
~~~ android
public class HelloAccessoryProviderConnection extends SASocket {

	// ...
	
	@Override
	public void onReceive(int channelId, byte[] data) {
		
		// for example
		Toast.makeText(getBaseContext(),
                new String(data), Toast.LENGTH_LONG)
                .show();		
	}
~~~

onReceive method is fired every time Android device receives data from Gear. You can even create a simple 'switch' here, using String or JSON objects. In the code above we have the easiest example - smartphone receives data from Gear and displays it as Toast notification. As simple as that... 

Wait, what if you want to send data from host-side to Gear? It's not explained in the tutorial app, but don't worry - I'm here to show you.

###Sending - Android
~~~ android
public class HelloAccessoryProviderConnection extends SASocket {

	// ...
	
	public void sendNotification(final String notification) {
	
		final HelloAccessoryProviderConnection uHandler = mConnectionsMap.get(mConnectionId);
					
		if(uHandler == null){
				Log.e(TAG,"Error, can not get handler");
				return;
			}
			
		new Thread(new Runnable() {
			public void run() {
				try {
					uHandler.send(HELLOACCESSORY_CHANNEL_ID, notification.getBytes());
				} catch (IOException e) {
					e.printStackTrace();
				}
			}
		}).start();
	
	}
~~~

That's how you send data to Gear from Android device. First of all, you get a uHandler from connectionsMap (this ensures that the connection with smartwatch is active). Then you simply post binary data on the specific channel using Thread. It looks easy, but the trickiest part is to properly launch this method when needed. As I stated at the beginning, SAAgent class extends Android Service. Then, the simplest, yet quite effective way to communicate with the Service (and our sendNotification method) is to use Broadcasts.

~~~ android
public class HelloAccessoryProviderService extends SAAgent {

	// ...

	public class HelloAccessoryProviderConnection extends SASocket {
		// ...
		
		@Override
		public void onReceive(int channelId, byte[] data) {
		
			// registering the BroadcastReceiver here ensures you, that the Gear connection has been already established
			GearDataReceiver gearDataReceiver = new GearDataReceiver();
			IntentFilter intentFilter = new IntentFilter("myData");
			registerReceiver(gearDataReceiver, intentFilter);
			
			
			// for example
			Toast.makeText(getBaseContext(),
					new String(data), Toast.LENGTH_LONG)
					.show();			
		}
	}
	
	
	// code below goes to the outer class, which extends SAAgent
	private class GearDataReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            if (intent.getAction().equals("myData")) {
				String data = intent.getStringExtra("data");
				notifyGear(data); 
            }
        }
    }
	
	public void notifyGear(String notification) {
		
        for(HelloAccessoryProviderConnection provider : mConnectionsMap.values()) {
            provider.sendNotification(notification);
        }
    }
}
~~~

Code shown above lets you send any data to Gear when HelloAccessoryProviderService receives a Broadcast. All you have to do is send that broadcast from an appropriate Activity:

~~~ android
// method to place in your Activity
public void sendDataToGear(View view) {
	Intent intent = new Intent("myData");
	intent.putExtra("data", "Hello Gear!");
	sendBroadcast(intent);
}
~~~

###Receiving - Gear
Now, when you know how to handle connection from the Android host part, it's the time to show you some basic Gear-side scenarios. It's much easier, so don't worry. Just remember, that we are coding in JavaScript here.

~~~ javascript
// use this code, when you are sure, that SASocket has been established
try {
		SASocket.setDataReceiveListener(onReceive);
} catch(err) {
		console.log("exception [" + err.name + "] msg[" + err.message + "]");
}


function onReceive(channelId, data) {
	// for example
	alert(data);
}
~~~

All you have to do is set an appropriate listener (remember, this thingy will work only with SASocket established), which drives the data received from Android. Told you it'd be simple.

###Sending - Gear
~~~ javascript
function myClick(data) {
	try {
		SASocket.sendData(CHANNELID, data);
	} catch(err) {
		console.log("exception [" + err.name + "] msg[" + err.message + "]");
	}
}
~~~
Another easy step. Just be sure to pass a correct argument to this method - it'll work flawlessly, if the SASocket has been established.

##Now it's your turn
That's it. It's time to code. You know how to handle the transferring data between Android and Gear. With that essential skill you are now able to develop almost every type of integrated app you want.

Feel free to leave a comment here or send me an email at adam@scalac.io - I am eager to help you. Have fun and play with the code above any way you want.
