---
layout: post
title: How to use Block.io for Wallet Creation and Webhooks on Parse
---

Over the weekend, I set out to build the backend elements for bitcoin wallet creation and notification of wallet balance changes using [Block.io](http://block.io). Block.io isn't as well-known or well publicized as it's competitors Chain, Gem.co and and others. But the clear distinction is that Block.io is the only provider to not abstract away what's happening?

Pros and Cons of Bitcoin APIs:

  | Chain | Block.io  | Gem |
------------- | ------------- |
Content Cell  | Content Cell  |
Content Cell  | Content Cell 

Now because I wanted to make the wallets easy to access for a future mobile app, I wanted to use Parse. Now there was a clear option to go the Node/Express route for the backend logic but a few blog posts recommended using Parse CloudCode. Parse CloudCode is essentially a node interpreter in the cloud with slightly unique syntax. 

You can code things like this, which invoke certain logic after a database save
{% highlight js %}
Parse.cloud.afterSave("_User"){
	// you can put any logic here
	console.log("The above logic gets invoked after we create and save a new User!");
}
{% endhighlight %}

There are other cool benefits like creating a job (think cron job...but in the cloud), which you can set to be recurring using a GUI. No more illegible and alien looking syntax to set the time interval of a cronjob to worry about.

{% highlight js %}
Parse.cloud.job("doNightly"){
	// insert database manipulation here
	console.log("This database manipulation will happen every night");
}
{% endhighlight %}

Bringing these elements together, I proposed a solution that went as follows.

1. User signs up via a UI and creates an account
2. An AfterSave CloudCode function is called and invokes an API call to Block.io
3. The API response contains a wallet public key which we store in a new class that points to the User
4. User is shown there public key and prompted to deposit
5. Block.io webhook is used to monitor changes in the (newly created) wallet and respond on change
6. Verify that we've received bitcoin and allow user to see a dashboard

Alright, with the steps of the solution created. Let's jump in.

#Step 1: Initialize Parse and save new User details to User Class

Parse does this strange thing where they have special classes and classes that you create. Parse special classes are denoted with an undescore in front. The most often used example of 
this is the "_User" class. Parse makes it easy to track user sessions with this class, just remember when you're doing queries that you need an underscore.

Okay, so here we're initializing Parse and saving a new user by grabbing input data from 
an input on an Angular form.

{% highlight js %}
//this can go anywhere
Parse.initialize("API-KEY", "JS-KEY");

// inside the view controller, we add ng-click="register()" to the submit button

$scope.register = function() {
	// we're holding the user input using Angular's $scope
   var name = $scope.user.name;
   var passWord = $scope.user.password;
   var email = $scope.user.email;

   // using Parse's special user class, we crate a new user
   var user = new Parse.User();

   //save to the new user their credentials
   user.set("username", name);
   user.set("password", passWord);
   user.set("email", email);
{% endhighlight %}

#Step 2: Create an AfterSave CloudCode Function for the User Class

We now jump into CloudCode. CloudCode is accessed by clicking on "Core" and is on the left sidebar. Again, CloudCode is all in javascript and has access to your classes. You can 
call APIs, do data manipulation, almost anything. Except for a few key areas, like npm packages and sockets. Both of these become frustrating if you get deep into CloudCode.

Back to our code, below is an AfterSave to call block.io and create a wallet. We're saving the response to a new class.

{% highlight js %}
//Make user a wallet.
Parse.Cloud.afterSave("_User", function(request) {
	//initiate API call to block.io, make sure to add API key
    Parse.Cloud.httpRequest({
      url: 'https://block.io/api/v2/get_new_address/?api_key=APIKEY',
      success: function (d) {
 				//parse the response from block.io
        var walletData = JSON.parse(d.text);
 				// these two lines act to select the Wallet class and make a new row
        var WalletClass = Parse.Object.extend("Wallet");
        var wallet = new WalletClass();
 				// now we grab the public key and assign it to the Wallet calss
        wallet.set("publicKey", walletData.data.address);
        wallet.set("owner", request.object);
        // and we save everything at the end
        wallet.save();
});
{% endhighlight %}

So to reiterate, we've create a user and instantly after saving their details to Parse. Have made an API call to create them a wallet. From the response from block.io, we're making a new row in our Wallet class to save the data.

#Step 2: Create a Webhook to Check for Changes in the Wallet

Our next step on the user interface was to prompt the user with a classic QR code and the public key (the one we just made). The idea here is that the user opens their bitcoin wallet provider (Coinbase, Circle, Snapcard, Breadwallet, etc) and sends money to this address. We want to know when they've sent that money and give them access to our app.

There is a much longer discussion about confirmations from the blockchain network to be had here and we certainly thought about it a lot. There are pros and cons of waiting for 3 confirmations (as we do and so does block.io) but is the topic of another blog post.


{% highlight js %}
// ...still inside the AfterSave Response
// adding a second API call to block.io to make a webHook and set a callback on CloudCode
Parse.Cloud.httpRequest({
   url: 'https://block.io/api/v2/create_notification/?api_key=6cc7-b07d-b22b-f6d2&type=address&address=' + walletData.data.address + '&url=https://9uH7wnXCApdzv4vtL38fN8w1F8YpWXzzqhx9ITtp:javascript-key%3DTi6naCnm6CVBjo5iAjljsV8ByduSBWnj2OSCF8ZL@api.parse.com/1/functions/transaction_received',
	   success: function (d) {
	   	console.log(d.text);
	     },
	   error: function () {
	      console.error('error' + d.text);
	     	}
	    });
	   },
	   error: function () {
	      console.error("not working properly");
	      }
    });

{% endhighlight %}


So that's the tutorial. Block.io is definitely not the ultimate best bitcoin API to work with althought in the context of CloudCode where you cannot access npm modules. It's the best bet to solve the problem set we had.
