# Django Paypal Payment Gateway Integration

This repo is meant as an extension to this article [Django Paypal Payment Gateway Integration with Working Example](https://studygyaan.com/django/django-paypal-payment-gateway-integration-tutorial) by Adam Smith.

While following the steps of the article, I faced some missing information and steps, so I decided to reproduced this tutorial as a minimal Django application that you can use as an working example and compare your own code to.

Further, this `README` is meant to help you guide through his described steps and set up the same code as shown in this repo.

# Step 1
- install virtual environment:

`python -m venv venv --prompt=venv` 

- activate virtual environment

`./venv/Scripts/activate`

- install Django within virtual environment

`pip install django`

- start Django project called 'django-paypal'

`django-admin startproject django_paypal`

- navigate into your project folder:

`cd .\django_paypal\`

- start Django app called 'paypal_app'

`django-admin startapp paypal_app `

- add 'paypal_app' to `INSTALLED_APPS` in `settings.py`

```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'paypal_app',
]
```

- install `paypalrestsdk` Python package to interact with the PayPal API

`pip install paypalrestsdk`

# Step 2: Set up Paypal Sandbox
- go to to https://developer.paypal.com/dashboard/applications/sandbox
- click on `create app` and create a new app for your Django application
- open your Paypal-Sandbox app and look for your Client Id and Secret
- in `settings.py` add the following two lines to the bottom:
```
PAYPAL_CLIENT_ID = 'your_paypal_client_id'
PAYPAL_SECRET = 'your_paypal_secret'
```
(Replace each string with the actual Client Id and Secret from your Paypal Sandbox)

Note: These should not be shared, e.g. on GitHub. Thus, you can create a new file called `credentials.py` in your project and place bothe lines there.

Then, import both variables to your `views.py`:
`from django_paypal.credentials import PAYPAL_SECRET, PAYPAL_CLIENT_ID`

Finally, updated the configuration part to:
```
paypalrestsdk.configure({
    "mode": "sandbox", 
    "client_id": PAYPAL_CLIENT_ID, # Updated
    "client_secret": PAYPAL_SECRET, # Updated
})
```

# Step 3: Create Views
Create three views:
- create_payment
- execute_payment
- payment_checkout
- payment_failed

```
# views.py

import paypalrestsdk
from django.conf import settings
from django.shortcuts import render, redirect
from django.urls import reverse

paypalrestsdk.configure({
    "mode": "sandbox",  # Change to "live" for production
    "client_id": settings.PAYPAL_CLIENT_ID,
    "client_secret": settings.PAYPAL_SECRET,
})

def create_payment(request):
    payment = paypalrestsdk.Payment({
        "intent": "sale",
        "payer": {
            "payment_method": "paypal",
        },
        "redirect_urls": {
            "return_url": request.build_absolute_uri(reverse('execute_payment')),
            "cancel_url": request.build_absolute_uri(reverse('payment_failed')),
        },
        "transactions": [
            {
                "amount": {
                    "total": "10.00",  # Total amount in USD
                    "currency": "USD",
                },
                "description": "Payment for Product/Service",
            }
        ],
    })

    if payment.create():
        return redirect(payment.links[1].href)  # Redirect to PayPal for payment
    else:
        return render(request, 'payment_failed.html')

def execute_payment(request):
    payment_id = request.GET.get('paymentId')
    payer_id = request.GET.get('PayerID')

    payment = paypalrestsdk.Payment.find(payment_id)

    if payment.execute({"payer_id": payer_id}):
        return render(request, 'payment_success.html')
    else:
        return render(request, 'payment_failed.html')

def payment_checkout(request):
    return render(request, 'checkout.html')

def payment_failed(request):
    return render(request, 'payment_failed.html')
```

# Step 4: Create Templates and URL’s
- in your app folder create a folder named 'templates'

## Create payment_success.html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Payment Successful</title>
</head>
<body>
    <h1>Payment Successful</h1>
    <p>Thank you for your purchase!</p>
</body>
</html>
```

## Create payment_failed.html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Payment Failed</title>
</head>
<body>
    <h1>Payment Failed</h1>
    <p>Your payment was not completed. Please try again.</p>
</body>
</html>
```

## Create a button intitate Paypal payment
```
<form action="{% url 'create_payment' %}" method="POST">
    {% csrf_token %}
    <button type="submit">Pay with PayPal</button>
</form>
``` 

## Add URL patterns for your views

In the `urls.py` within your project folder import your `views.py`:

`from paypal_app import views`

Include the new urls:
```
urlpatterns = [
    ...
    path('checkout/', views.payment_checkout, name='checkout_payment'),
    path('create_payment/', views.create_payment, name='create_payment'),
    path('execute_payment/', views.execute_payment, name='execute_payment'),
    path('payment_failed', views.payment_failed, name='payment_failed')
]
```

# Step 5: Start Server
Start your app:

`python manage.py runserver`

*Note: Since we haven defined a default url (for `http://127.0.0.1:8000/`) we will get an error:
`Page not found (404)`. This is not a problem.*

Navigate to:
`http://127.0.0.1:8000/checkout/`

The checkout page will look like this:

![checkout-page](./checkout-page.jpg)

When clicking on `Pay with Paypal`, we are asked to login into Paypal.

*Note: Check the url. You will see that we are asked to log into the Sandbox-Paypal account.*

Go to https://developer.paypal.com/dashboard/accounts and click on the account with `...@personal.example.com`

Use the email and password to log into the Sandbox account.

Confirm the payment. You should be redirected to your `payment_success.html` page.

# Step 6: Confirm payment

Go to: https://sandbox.paypal.com

Log in using the business Sandbox account

(Go to https://developer.paypal.com/dashboard/accounts and click on the account with `...@business.example.com`)

Check if the payment occurs in your recent activity.

