# Building a WhatsApp-based QR & Bar code scanner

After realizing that you could send and receive images on your WhatsApp bot with twilio, I kept thinking of ways I could exploit this feature, and make something useful out of it. But then, the question was, **what** ? The enlightenment came when I was paying for groceries at a store. What if there was a way to automate the barcode reading process using a WhatsApp bot? ( *beard scratching thinking*)

In this article, we are going to be creating a code scanner which can convert any barcode or qr code sent to it, into text. 



### Prerequisites

Before we start writing any code, we have to make sure that the following dependencies are met. 

- Python is installed on your machine. Since the application is written in Python, it makes sense to have it installed, right?
- Visual C++ 13 redistributables are installed. If not, download and install them from [ https://www.microsoft.com/en-US/download/details.aspx?id=40784](https://www.microsoft.com/en-US/download/details.aspx?id=40784). This is critical because most likely it is not installed on most Windows 10 PC's and will lead to a `ImportError` or a `DecodeError`.
- OpenCV: preferably should be pre-installed if an anaconda distribution was used. If not however, should still work fine if pip installed as long as you have Visual C++ 15 distribution installed.
- A smartphone with an active phone number and WhatsApp installed.
- A twilio account.



## Configure the Twilio WhatsApp Sandbox

Twilio provides a [WhatsApp sandbox](https://www.twilio.com/console/sms/whatsapp/learn) where you can easily develop and test your application. Once your application is complete you can request [production access for your Twilio phone number](https://www.twilio.com/whatsapp/request-access), which requires approval by WhatsApp.

Let’s connect your smartphone to the sandbox. From your [Twilio Console](https://www.twilio.com/console), select [Programmable SMS](https://www.twilio.com/console/sms/dashboard) and then click on [WhatsApp](https://www.twilio.com/console/sms/whatsapp/learn). The WhatsApp sandbox page will show you the sandbox number assigned to your account, and a join code.

![](https://twilio-cms-prod.s3.amazonaws.com/images/PTXUsKgR_iJOTf_c-6BJ6RSwJl6Dtp-rF45Tgai7pGLHCz.width-500.png)

To enable the WhatsApp sandbox for your smartphone send a WhatsApp message with the given code to the number assigned to your account. The code is going to begin with the word **join**, followed by a randomly generated two-word phrase. Shortly after you send the message you should receive a reply from Twilio indicating that your mobile number is connected to the sandbox and can start sending and receiving messages.

Note that this step needs to be repeated for any additional phones you’d like to have connected to your sandbox.



## Project setup

In this section we are going to set up a new Flask project. To keep things organized, open a terminal or command prompt, navigate to your favorite directory for projects and type the following commands:

UNIX ( Linux or MacOS)

```shell	
mkdir whatsapp-code-scanner
cd whatsapp-code-scanner
python3 -m venv venv
source venv/bin/activate
```

Windows

```batch
md whatsapp-code-scanner
cd whatsapp-code-scanner
python -m venv venv
venv/Scripts/activate
```

So, what do the above commands do? 

To put it simply, they create a virtual environment, and then activate the newly created environment. These are the best practices, highly recommended by experts everywhere ... at least the Python experts.

The next thing to do, is to install the project requirements. In this project, we need Pyzbar, OpenCV, Flask, requests, and of course, twilio.

```shell
(venv) $ pip install flask twilio opencv-python requests pyzbar numpy
```



### Let's Code!

Once the dependencies have been installed, open the directory with the project in your favorite editor (VSCode obviously), create a file called *app.py* and open it in the editor. 

```python
import logging
import cv2 as cv
import numpy as np
import requests
from twilio.twiml.messaging_response import MessagingResponse
from flask import Flask, request
from pyzbar import pyzbar

app = Flask(__name__)

@app.route('/')
def test():
    return "This works!"
```

What we have done so far is to simply import the dependencies, and initialize our flask app. Nothing fancy ... well ... not yet anyways.

The next step, is to create the WhatsApp webhook. Twilio uses the concept of [webhooks](https://www.twilio.com/docs/usage/webhooks) to enable your application to perform custom actions as a result of external events such as receiving a message from a user on WhatsApp. A webhook is nothing more than an HTTP endpoint that Twilio invokes with information about the event. The response returned to Twilio provides instructions on how to handle the event.

The webhook for an incoming WhatsApp message will include information such as the phone number of the user and the text of the message. In the response, the application can provide a response to send back to the user. The actions that you want Twilio to take in response to an incoming event have to be given in a custom language defined by Twilio that is based on XML and is called [TwiML](https://www.twilio.com/docs/voice/twiml).

With the fancy explanations out of the way, let's get back to coding. Open the *app.py* file and add the following code:

```python
@app.route("/messages", methods=['GET', 'POST'])
def message_reply():
    """Respond to incoming calls with a simple text message."""
    # First log the incoming request 
    logging.warning(request.values)
    # Start our TwiML response
    # Create response object
    resp = MessagingResponse()
    # Get the message received by our Twilio number
    body = request.values.get('Body', None)
    if body:
        # Add your response logic to respond to text based requests
        resp.message("received request!")
        return str(resp)

    # Check if media received was an image
    media = request.values.get('MediaContentType0', None)
    if 'image' in media:
        img_url = request.values.get('MediaUrl0') # get the url of the image
        img_raw = requests.get(img_url).content
        img_data = np.asarray(bytearray(img_raw), dtype="uint8")
        img = cv.imdecode(img_data, cv.IMREAD_COLOR)
        if img.size > 0:
            res = pyzbar.decode(img)
            try:
                code_data = res[0].data
                code_type = res[0].type
                if code_type != 'QRCODE':
                    code_type = 'BARCODE'
                # Add your response logic here to send a message, here it's just "hello world"
                resp.message(f"Your {code_type} is {code_data}")
            except IndexError:
                resp.message("No Code Detected In Image.")
        else:
            resp.message("Image couldn't be read/parsed")
    else:         
        # Add your response logic here to send a message, here it's just "hello world"
        resp.message("Sorry, could you try sending your message again?")
    return str(resp)
```

When a WhatsApp message is sent by the user to the sandbox number, Twilio forwards the message to the web-hook (the `/messages` endpoint created above), as a request object. We first check if the message sent contained an image, since someone might send the bot text or video instead of an image. After successfully validating that the message contains an image, we then get the image by using the `requests` module.

We then convert the image to a `numpy` array, and then use `opencv` to get the patterns on the image. From the patterns, we then use pyzbar to decode what they actually mean.

It's a very simple process, really. In the end, we then serve the application by adding the following code in the *app.py*:

```python
if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0")
```



## Testing the application

Great! Now that we're done-ish, we can now test to see if our application works. Run the following commands:

Unix

```shell
export FLASK_APP=app.py
flask run
```

Windows (Command Prompt)

```powershell
set FLASK_APP=app.py
flask run
```

{Image goes here}

Now you have a web application running locally on your computer. However, in order for twilio to communicate with our application, it needs to be available on the internet and during development, we can use the [ngrok](https://ngrok.com/) utility to create a temporary public URL for our application.

First download `ngrok` from [here](https://ngrok.com/download) and then when you have it, run:

Unix

```shell
./ngrok http 5000
```

Windows (Command prompt)

```shell
ngrok.exe http 5000
```

Note the lines beginning with “Forwarding”. These show the public URL that ngrok uses to redirect requests into our local application. Take either one of these URLs and enter it on your browser. Ngrok will randomly generate the first part of the URL every time it starts.

{Ngrok image goes here}



To connect the webhook with WhatsApp you need to configure its URL in the Twilio console. Locate the [WhatsApp sandbox settings](https://www.twilio.com/console/sms/whatsapp/sandbox) page and edit the “When a message comes in” field with the URL of your webhook. This is going to be the temporary ngrok URL with */message* appended at the end. You can see an example below:

![](https://twilio-cms-prod.s3.amazonaws.com/images/cCIcmMBznujuXAFcY7sMbvN3a2QRn_QUAJ4aUzMGNiZTsX.width-500.png)

Make sure the dropdown to the right of the URL field is set to “HTTP Post”, and don’t forget to click the “Save” button at the bottom of the page to record these changes.

After configuring, we can test our application by sending an image.

{Image goes here}
