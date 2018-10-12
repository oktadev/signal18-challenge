# Twilio Signal 2018 DEVELOPER CHALLENGE!

Hello, friend! I'm glad you ran into us at Signal!

If you'd like to get an Okta prize, follow the instructions below. I'll walk you
through building a very simple serverless app with Twilio and Okta.


## Introduction

You (obviously) know what Twilio is, since you ran into us at Signal and all,
but you may not be familiar with Okta.

We are an API service that makes it easy to store user accounts for the
applications you're building. We make it easy to handle things like:

- User registration
- User login
- Password reset
- API authentication
- Authorization
- Etc.


## What You Will Build

To complete the challenge you need to:

1. [Sign up for a free Okta developer account](https://developer.okta.com/signup/)
2. [Sign up for a Twilio account](https://www.twilio.com/try-twilio) (if you don't already have one)
3. Follow the instructions below and get your application functional. Once it is
   working, show it to an Oktanaut at the Okta booth to claim your prize!

The project you'll be building is a simple business phone system.

Here's the scenario: you're a developer (or IT person) working at a company and
you have several employees. All of your employee accounts are stored in Okta
(this is a common real-world scenario). When new employees join your company,
they are asked to specify their cell phone number in their company profile
(stored in Okta) so other employees can contact them if they need to.

Your company is now big enough that you'd like to get each employee a dedicated
company phone number to put on their business cards, etc. You want to find a
simple and cost effective way to give each employee a dedicated phone number
that simply forwards calls and SMS messages from the company number to each
employee's personal cell phone. This way, employees can still use their cell
phones but have a company number.

To accomplish this, you'll use Okta to store your employee accounts (and phone
numbers) and Twilio to purchase phone numbers and handle call and SMS forwarding
functionality.


## Set Up Okta

Once your Okta account is created, complete the following tasks.

1. Copy down your `Org URL` from your Okta dashboard page.
2. Visit the **API** -> **Tokens** page and create a new token. This will be
   your Okta API key that you'll use to work with the service. Copy this value
   down someplace for later usage.
3. Visit the **Users** -> **People** page and click **Add Person**. Create a new
   user (a fake employee). Click on the newly created user account then click on
   the **Profile** tab. Click **Edit** to edit the user's profile and enter your
   cell phone number into the **Mobile phone** field, then click **Save** to
   save your phone number into this user account.

That's it for Okta setup! You now have a fake employee account and your employee
account has a real cell phone number in it's profile. This will be used later
on.


## Set Up Twilio

Once your Twilio account is created, complete the following tasks.

1. Copy down your **Account SID** and **Auth Token** values from the [Twilio
   Dashboard](https://www.twilio.com/console). These are your API credentials
   that you need to talk to Twilio's API.
2. Visit the [Twilio Functions Page](https://www.twilio.com/console/runtime/functions/manage) and create a
   new function. When prompted, select **Blank** as your template. Use the
   following settings to define your function then click **Save**.
   - **Function Name**: Call Forward
   - **Path**: /call-forward
   - **Check for valid twilio signature**: CHECK
   - **Event**: Incoming Voice Calls
   - **Code**: (below)
     ```javascript
     const okta = require("@okta/okta-sdk-nodejs");
     const MemoryStore = require("@okta/okta-sdk-nodejs/src/memory-store");

     exports.handler = function(context, event, callback) {
       const oktaClient = new okta.Client({
         orgUrl: process.env.OKTA_ORG_URL,
         token: process.env.OKTA_TOKEN,
         requestExecutor: new okta.DefaultRequestExecutor(),
         cacheStore: new MemoryStore({ keyLimit: 100000, expirationPoll: null })
       });
       let user;

       oktaClient.listUsers({
         search: 'profile.primaryPhone eq "' + event.To + '"'
       }).each(u => {
         user = u;
       }).then(() => {
         let twiml = new Twilio.twiml.VoiceResponse();

         twiml.say("Please wait. You are being connected to " + user.profile.firstName + ".");
         twiml.dial({
           callerId: event.From ? event.From : undefined,
           answerOnBridge: true
         }, user.profile.mobilePhone);
         twiml.say("Goodbye.");
         callback(null, twiml);
       });
     };
     ```
3. Visit the [Twilio Functions Page](https://www.twilio.com/console/runtime/functions/manage) and create a
   new function. When prompted, select **Blank** as your template. Use the
   following settings to define your function:
   - **Function Name**: SMS Forward
   - **Path**: /sms-forward
   - **Check for valid twilio signature**: CHECK
   - **Event**: Incoming Messages
   - **Code**: (below)
     ```javascript
     const okta = require("@okta/okta-sdk-nodejs");
     const MemoryStore = require("@okta/okta-sdk-nodejs/src/memory-store");

     exports.handler = function(context, event, callback) {
       const twilioClient = context.getTwilioClient();
       const oktaClient = new okta.Client({
         orgUrl: process.env.OKTA_ORG_URL,
         token: process.env.OKTA_TOKEN,
         requestExecutor: new okta.DefaultRequestExecutor(),
         cacheStore: new MemoryStore({ keyLimit: 100000, expirationPoll: null })
       });
       let user;

       oktaClient.listUsers({
         search: 'profile.primaryPhone eq "' + event.To + '"'
       }).each(u => {
         user = u;
       }).then(() => {
         twilioClient.messages.create({
           to: user.profile.mobilePhone,
           from: event.To,
           body: "From: " + event.From + "\n\n" + event.Body
         }, (err, message) => {
           callback();
         });
       });
     };
     ```
