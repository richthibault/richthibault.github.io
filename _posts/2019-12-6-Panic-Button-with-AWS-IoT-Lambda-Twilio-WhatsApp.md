---
layout: post
title: Creating a Panic Button with AWS IoT, AWS Lambda, Twilio, and WhatsApp
tags:
 - AWS
 - IoT
 - Lambda
 - Twilio
 - WhatsApp
---

(NOTE: This article is few years old now, and many of the links no longer work!)

A family member has been having some health issues lately, so I thought it might be helpful for her to have sort of a "panic button" she can press if she's feeling ill and needs help.  The button would need to be simple to use, and should send a message to nearby relatives so they can run and assist.  I also wanted it to use WhatsApp, since she lives in Colombia, where WhatsApp is more popular than SMS.

So first I would need a button.  I looked at some Raspberry Pi solutions, but those seemed overly complicated.  Then I stumbled across the AWS IoT Enterprise button:

<a href="https://www.amazon.com/All-New-AWS-IoT-Enterprise-Button/dp/B075FPHHGG" target="_blank">https://www.amazon.com/All-New-AWS-IoT-Enterprise-Button/dp/B075FPHHGG</a>

This seemed like a great solution - an easy-to-use button, small and portable, and I could write the supporting code in Java, deploying to AWS Lambda to handle the button clicks.  And at only 20 bucks, it’s affordable as well.

So here’s what we need to build the solution:

* AWS IoT Enterprise button
* Java
* Twilio API for Java
* Amazon Web Services account for AWS Lambda

**Important Note:** The IoT Enterprise button requires Wifi to function - so this won’t be a portable panic button, it will be mounted in the house.  AT&T offers an LTE button (https://marketplace.att.com/products/att-iot-button-sercomm-332372) that works on the cellular network, if portability is what you need.

## Purchasing and Setting Up AWS IoT

The easy part is buying the button - just go to:

https://www.amazon.com/All-New-AWS-IoT-Enterprise-Button/dp/B075FPHHGG

While you’re waiting for it to arrive, be sure to have an AWS account ready to go:

https://aws.amazon.com

You’ll want to go ahead and put in your credit card in the billing section (My Account - Payment Methods).  This may seem unnecessary given that AWS Lambda gives us 1 million free requests per month, which is about 999,999 more than I need.  But I couldn’t get the button to activate until I set up a credit card in my account.  (AWS does charge a minimal fee for having an active, connected IoT button.)

Once the button shows up on your doorstep, then you can register it here:

https://aws.amazon.com/iot-1-click/

Click “Get Started with AWS IoT 1-Click” to begin. The following page will have links to install the "AWS IoT 1-Click” app, so install that first on your iOS or Android device.  From there, follow the instructions in the app to get the device onto your Wifi network.

Then go back to the AWS Console and click “Claim Devices.” I couldn’t find a device ID on the device itself, but checking the Order Details page on Amazon, I found a Serial Number.  It’s 16 characters and starts with G030.  I copied and pasted that in as the claim code, clicked “Claim,” and then the button was tied to my AWS account.  So far so good.

If you only want to send an SMS, you are almost done - create a project from the IoT 1-Click page, and give it a name and description.  Then click “Start” under Device Templates.  Click “All button types,” enter a Device Template Name, and choose Send SMS.  Enter your phone number and the SMS text, then Create Project.  (You can also choose to Send Email if you prefer.)  Then click “Create Placements,” enter a Placement Name, and select your registered device. Click “Create Replacement” and you should be all set.

If you want to send a WhatsApp message like I did, read on.

## Setting Up Twilio

Twilio provides an API for Internet-based telephony services, so it’s a great way of sending SMS or WhatsApp messages from a Java application.  Sign up for a developer account here:

https://www.twilio.com/

I find just having a trial developer account works great for the purposes of this app, which is very low traffic (user base of 1!).  Sign up for a free trial account, then access the Programmable SMS service in the Twilio console.  Generate your API credentials, Account SID and Auth Token - you will need these later.

To use WhatsApp, go to the WhatsApp Beta section of the Twilio console to set up your sandbox.  You’ll need to use WhatsApp on your phone to send a message to the predefined Twilio number shown on the Learn page, and join using the two-word code shown for your sandbox: “join xxxx-yyyy”.  Now your phone is available to receive messages on WhatsApp via Twilio.  Each planned recipient of your app’s messages will need to do the same thing.

If you’re doing a project that will involve more volume, you can request a production account, which has to go through an approval process.  But again, for my simple needs, the trial account works just fine.

Now that we’re all set up, let’s do some coding.

## Developing the Lambda Function in Java

I use Maven for dependencies, so I create a new Maven project in Eclipse.  We’ll need the Twilio API and AWS Lambda as dependencies, so the dependencies in our pom.xml will look like this:

{% highlight xml %}
    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-lambda-java-core</artifactId>
        <version>1.2.0</version>
    </dependency>
    
    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-lambda-java-events</artifactId>
        <version>2.2.7</version>
    </dependency>
    
    <dependency>
        <groupId>com.twilio.sdk</groupId>
        <artifactId>twilio</artifactId>
        <version>7.45.1</version>
        <scope>runtime</scope>
    </dependency>
{% endhighlight %}

Having aws-lambda-java-events gives us access to the IoTButtonEvent class, which is what gets triggered by button presses.  You can probably get by without this, but it’s helpful if you want to determine if the button was single-pressed, double-pressed, or long-pressed.  

Let’s create an implementation for the interface com.amazonaws.services.lambda.runtime.RequestHandler:

{% highlight java %}
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.IoTButtonEvent;

public class PanicButtonRequestHandler implements RequestHandler<IoTButtonEvent,Void> {

    public Void handleRequest(IoTButtonEvent event, Context context) {
        
        return null;

    }
}
{% endhighlight %}

The generic types provided to the RequestHandler are the aforementioned IoTButtonEvent, so we get access to the button click event, and Void, since I am just returning null.  (RequestHandlers can return Integers and other values, for use on other types of IoT devices, but we won’t need those for our simple button.)

Now let’s specify our Twilio Account SID, Auth Token, the “from” phone number, and the “to” phone number.  The phone numbers must follow a specific formatting - the “whatsapp:” protocol, followed by + and country code (1 for US), and then the number, with no spaces or dashes.  So for (305) 123-4567, the format would be:

whatsapp:+13051234567

The “from” number would be the same number to which you sent the join message - it’s a Twilio number, most likely with 415 area code.  The “to” number should be your own phone number, at least for now while we’re testing.

We also specify the message to send via WhatsApp.  When using a trial sandbox, there are very specific text templates you have to use - no spamming allowed!  The template I am using is this:

Your {} code is {}

It won’t win any Nobel prizes for literature, but I should be able to get the point across with this message:

Your *health* code is *SENDHELP*

Using a text string that doesn’t follow the template will result in a message send failure.

So we’ll plug all these variables in, initialize Twilio with our account credentials, then create a message to send it.

{% highlight java %}
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.IoTButtonEvent;
import com.twilio.Twilio;
import com.twilio.rest.api.v2010.account.Message;
import com.twilio.type.PhoneNumber;

public class PanicButtonRequestHandler implements RequestHandler<IoTButtonEvent,Void> {

    public Void handleRequest(IoTButtonEvent event, Context context) {
        
        String twilioSid = "<ACCOUNT_SID>";
        String authToken = "<AUTH_TOKEN>";
        
        String fromPhone = "whatsapp:+14159999999”;  // Twilio from number
        String toPhone = "whatsapp:+19999999999”;      // Recipient’s to number
        String msg = "Your health code is SENDHELP";
        
        PhoneNumber from = new PhoneNumber(fromPhone);
        PhoneNumber to = new PhoneNumber(toPhone);
        
        Twilio.init(twilioSid, authToken);
        Message message = Message.creator(to, from, msg).create();
            
        Integer errorCode = message.getErrorCode();
        if(null!=errorCode)
            System.out.println(String.format("Got Twilio error %d, %s.", errorCode, message.getErrorMessage()));
        
        return null;

    }
}
{% endhighlight %}

Let’s write a simple unit test to make sure messages are coming through, then run it:

{% highlight java %}
public class PanicButtonRequestHandlerTest {
    
    private PanicButtonRequestHandler panicButton;
    
    @Before
    public void setup() {
        panicButton = new PanicButtonRequestHandler();
    }

    @Test
    public void testClick() throws Exception {
        IoTButtonEvent event = new IoTButtonEvent();
        panicButton.handleRequest(event, null);
    }

}
{% endhighlight %}

Hopefully you just got a WhatsApp message on your phone!  If not, hopefully you got a helpful error message.  You probably just need to make sure you have your sandbox and phone set up properly in the Twilio console, and that the phone numbers are correct and formatted properly.

Once we’re sure it’s working, we can iterate a bit to improve things.  I moved all the variables into environment variables, so I can make the code reusable - no worries, we can use all the environment vars we want when we post this to AWS Lambda.  For purposes of unit testing, I clicked Debug - Debug Configurations in Eclipse, then entered all my environment vars on the Environment tab for my unit test.

{% highlight java %}
    private static final String TWILIO_SID_VAR = "TWILIO_SID";
    private static final String TWILIO_AUTH_TOKEN_VAR = "TWILIO_AUTH_TOKEN";

    private static final String PANIC_BUTTON_FROM_VAR = "PANIC_BUTTON_FROM";
    private static final String PANIC_BUTTON_TO_VAR = "PANIC_BUTTON_TO";
    private static final String PANIC_BUTTON_MSG_VAR = "PANIC_BUTTON_MSG";

    public Void handleRequest(IoTButtonEvent event, Context context) {
    	
    	String twilioSid = System.getenv(TWILIO_SID_VAR);
    	String authToken = System.getenv(TWILIO_AUTH_TOKEN_VAR);
    	
    	if(null==twilioSid || null==authToken)
    		throw new RuntimeException(String.format("No Twilio credentials found, set environment variables for %s and %s.", TWILIO_SID_VAR, TWILIO_AUTH_TOKEN_VAR));

    	String fromPhone = System.getenv(PANIC_BUTTON_FROM_VAR);
    	String toPhone = System.getenv(PANIC_BUTTON_TO_VAR);
    	String msg = System.getenv(PANIC_BUTTON_MSG_VAR);
    	
    	if(null==fromPhone || null==toPhone || null==msg)
    		throw new RuntimeException(String.format("No phone numbers found, set environment variables for %s, %s, and %s.", PANIC_BUTTON_FROM_VAR, PANIC_BUTTON_TO_VAR, PANIC_BUTTON_MSG_VAR));

        Twilio.init(twilioSid, authToken);
        ....
{% endhighlight %}

I also wanted to allow multiple "to" phones, so I set them as a comma-delimited list in the environment, then parse them into an array, and loop over each phone:

{% highlight java %}
        PhoneNumber from = new PhoneNumber(fromPhone);
        String toPhone = System.getenv(PANIC_BUTTON_TO_VAR);
        List<String> toPhones = Arrays.asList(toPhone.split("\\s*,\\s*"));
        
        for(String tp : toPhones) {
            PhoneNumber to = new PhoneNumber(tp);
            System.out.println(String.format("Sending message '%s' from %s to %s", msg, fromPhone, tp));
            Message message = Message.creator(to, from, msg).create();
            Integer errorCode = message.getErrorCode();
            if(null!=errorCode)
                System.out.println(String.format("Got Twilio error %d, %s.", errorCode, message.getErrorMessage()));
        }
{% endhighlight %}

For the full source code, see my Github page:

https://github.com/richthibault/panicbutton

Finally, we will need to create a jar for uploading to AWS Lambda.  My initial Lambda attempts failed due to missing classes - so we need to make sure we can generate a jar that includes the Twilio dependencies.  Add this to pom.xml:

{% highlight xml %}
  <build>
    <plugins>
      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
        </configuration>
      </plugin>
    </plugins>
  </build>
{% endhighlight %}

Now we can do “mvn package” and get the jar we want to upload to AWS. 

## Setting Up AWS Lambda

Log into the AWS console and go to the Lambda section, then click “Create Function.”  Choose to Author from Scratch, and select Java 8 or Java 11 for Runtime and click “Create Function.”  Upload your jar file, and enter the Handler, which should be something like:

com.yourpackage.PanicButtonRequestHandler::handleRequest

<img src="{{ site.baseurl }}/images/lambda-setup.png" alt="AWS Lambda Setup" style="width: 800px; height:391px;"/>

You can also enter all your environment variables, if using them. Click “Save” when done.  

Go back to the IoT 1-Click section of the AWS Console, and under Manage - Projects, click “Create.”  Give the project a Name like PanicButton, click “Next,” then click “Start” under Device Templates.  Click “All button types,” enter a Device Template Name, and choose Select a Lambda Function.  Select your Region and Lambda Function below, then click “Create Project."  Then click “Create Placements,” enter a Placement Name, and select your registered device. Click “Create Replacement” and you should be all set.  Now you can click the button to run your function.  If all is set up properly, you should get a WhatsApp message on your phone.

If it doesn’t work, let’s go to the logs.  Under the Lambda section, edit your lambda and go to the Monitoring section.  Click “View logs in Cloudwatch” to see any exceptions or other logged messages, and fix as necessary.

Hopefully you find this example helpful.  If you create a button of your own, hit me up in the comments and let me know how it goes.

## Update 1

I landed in Colombia, all set to install my nifty new button.  I figured all I need to do is fire up the IoT 1-Click app, tap Wifi Configuration, and get the device connected to the local Wifi network.  Wrong!  The app detected I was in Colombia, and threw an error: “This device is not supported in your location.”  Argh!  Panic!  I wasn’t about to let this roadblock stop me though.  I borrowed another family member’s Android phone, and searched for the IoT 1-Click app in the Google Play store - but of course it’s not available in Colombia.  So I found a website with the IoT 1-Click APK file, downloaded and installed it, authenticated on AWS, claimed the device again, and connected to the Wifi.  All set!

## Update 2

I finally got a bill for $.13 from AWS, for a partial month of having an active IoT button.  Next month the bill should be $.25.  Pretty good deal!
