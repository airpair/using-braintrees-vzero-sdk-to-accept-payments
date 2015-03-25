Braintree is a payment processing company that provides a very sensible API
for processing those payments. They provide a RubyGem called `braintree` and
they have released a JavaScript SDK called v.zero. The combination of these
two things -- the gem and the SDK -- allows you to accept payments very easily
from your customers in your Rails application. 

In this guide, I'll show you how to do exactly that. The best part is that no
sensitive credit card data touches your application; it all goes through
Braintree. Another good reason to use v.zero is that it seamlessly supports
accepting payments through PayPal.

Let's get started!

## Application setup

Here's an app I've prepared earlier: https://github.com/radar/braintree_shop_demo. This is the application we'll be using for this tutorial. Clone it down and set it up with these commands:

```
git clone git@github.com:radar/braintree_shop_demo
cd braintree_shop_demo
bundle install
```

When you start the app with `rails s` and go to http://localhost:3000, you'll see a very basic form:

![Initial checkout page](braintree/initial_checkout_page.png)

This is the form that we'll be using with Braintree's v.zero SDK and their gem.

## Adding v.zero to the form

Our first task is going to be adding v.zero to this form. We can do this by opening `app/views/home/index.html.erb` and adding this line to the bottom of the page:

```
<script src="https://js.braintreegateway.com/v2/braintree.js"></script>
```

This loads Braintree's JavaScript and provides a `braintree` object that we can use to setup v.zero in this form. We can do that by adding this content underneath the `<script>` tag that we just added:

```
<script>  
braintree.setup(
  "CLIENT_TOKEN_GOES_HERE",
  'dropin', {
    container: 'dropin'
});
</script>
```

This tells the Braintree JS SDK that we want to create a new credit card form
and put it in an element with the id of `dropin`. We don't have one of these
at the moment, but it's easy enough to add it to our form. Add it now after
the `last_name` field in the form:

```
<p>
  <%= label_tag "last_name" %><br>
  <%= text_field_tag "last_name" %>
</p>
<div id="dropin"></div>
```

The other thing that we'll need to do is replace the text of `CLIENT_TOKEN_GOES_HERE` with a real client token. These tokens are used by Braintree to verify that the requests that are being made to their API are legitimate. For this, we'll need to use the Braintree gem. The Braintree gem can be used to generate a client token by calling this code in `HomeController`'s `index` action:

```
def index
  @client_token = Braintree::ClientToken.generate
end
```

We can then use this token back in `app/views/home/index.html.erb`:

```
<script>  
braintree.setup(
  "<%= @client_token %.",
  'dropin', {
    container: 'dropin'
});
</script>
```

However things aren't entirely that simple. We'll need to install the Braintree gem and setup a Sandbox account before we can continue.

## Installing and configuring the Braintree gem

To add the Braintree gem to our application, we can add this line to our `Gemfile`:

```
gem 'braintree'
```

Then we can install it with `bundle install`. 

The next part is not as easy: we need to setup a Braintree sandbox account and get some configuration from that process. To sign up for a Braintree Sandbox account, go to https://www.braintreepayments.com/get-started and fill out the form. Once you're done there, jump over to your inbox and activate your account. Fill in this other short form, too.

Now you're inside your Braintree sandbox account. The configuration that we need for our client token generation to work can be found in the "Settings" -> "Users and Roles" menu near the top right of the page. From here, click on your username and scroll down to "API Keys". You should now see a screen that looks like this:

![Braintree keys](braintree_keys.png)

Click "View" underneath "Private Key" on the one key listed there and then you'll be able to see the configuration settings we're after. Except they're in Java! Not to worry! If you click on "Java", it'll bring up a menu of languages to choose from:

![Braintree keys menu](braintree_keys_menu.png)

Select "Ruby" from this menu. Now we finally have what we need! Copy the content of this box from Braintree and put it into a new file in the application called `config/initializers/braintree.rb`:

```
Braintree::Configuration.environment = :sandbox
Braintree::Configuration.merchant_id = '...'
Braintree::Configuration.public_key = '...'
Braintree::Configuration.private_key = '...'
```

I'm not going to show you my values here because they're private! You'll have
your own to keep private too.

Now that we have setup Braintree in our application, let's restart the `rails
server` process and see the fruits of our labor. Go back to
http://localhost:3000 and you'll now see the form.

![Braintree payment form](braintree_vzero_form.png)

We've got some new fields on our page: a card number and an expiry! There's
also a PayPal button there. This is the client-side of the v.zero API working.
Users can now enter their credit card details or choose to pay through PayPal.

These fields aren't a part of our initial form. Instead, Braintree's
JavaScript will encrypt these values safely and submit them to Braintree's
server. Braintree will then return a value called a _payment nonce_, which is
what we can use to capture the payment.

If we try to submit this form now with the first name, last name and credit
card fields filled out, we'll see this error:

![No route for checkout](no_route_for_heckout.png)

Now's the time to implement that route.

## Processing payments with Braintree

Now we get to the meat of what we're doing here: processing payments with Braintree.

Our first step will be to actually define the route for `POST /checkout` in
`config/routes.rb`. We can do this by adding this route to the bottom of our routes:

```
Rails.application.routes.draw do
  root to: "home#index"

  post "/checkout", to: "orders#checkout"
end
```

We're going to be sending this to a new controller because having this logic
in the `HomeController` really confuses what that controller does. 

Let's generate this new controller now:

```
rails g controller orders
```

Inside this controller, let's add the `checkout` action:

```
class OrdersController < ApplicationController
  def checkout
    result = Braintree::Transaction.sale(
      :amount => "10.00",
      :payment_method_nonce => params[:payment_method_nonce]
    )
    if result.success?
      flash[:notice] = "Payment was successful"
    else
      # TODO
    end
  end
end
```

The `Braintree::Transaction.sale` method used here takes an amount and the `payment_method_nonce` from our form and will tell Braintree that we want to charge the user's card $10USD. If that result is successful, then we'll show the checkout view with a `flash[:notice]` that tells the user that the payment was successful. We don't have anything for the unsuccessful case, but we will shortly.

Before we run through a successful checkout, we'll need to create the view at `app/views/orders/checkout.html.erb`. Put this content in it:

```
<h1>Thanks for buying my thing!</h1>

<strong>You've paid $10 for it.</strong>
```

Let's run through a successful checkout now. Go to http://localhost:3000 and fill in all the fields. For the credit card number, use the test card "4242 4242 4242 4242" with an expiry date in the (near) future. When you're done here you'll see the checkout page:

![Checkout page](checkout_page.png)

Great! We're now processing transactions through Braintree successfully.

To see one fail, we'll need to adjust our call to `Braintree::Transaction.sale` to use an amount higher than $2,000.

```
class OrdersController < ApplicationController
  def checkout
    result = Braintree::Transaction.sale(
      :amount => "2001.00",
      :payment_method_nonce => params[:payment_method_nonce]
    )
    ...
```

In Braintree's sandbox, any amount between $2,000 and 2,062.99 will return a different error depending on the dollar amount. These errors correspond with the ones listed in [Braintree's "Processor Response Codes" documentation](https://www.braintreepayments.com/docs/ruby/reference/processor_responses). By using $2,001 as the price, Braintree will tell us that the card has insufficient funds. We'll then grab this message from Braintree and display it to the user. Let's update the code again in the `checkout` action to do that:

```
class OrdersController < ApplicationController
  def checkout
    result = Braintree::Transaction.sale(
      :amount => "2001.00",
      :payment_method_nonce => params[:payment_method_nonce]
    )
    if result.success?
      flash[:notice] = "Payment was successful"
    else
      flash[:error] = result.message
      redirect_to home_path
    end
  end
end
```

The `result.message` is the key here. That'll give us the error message from Braintree which we'll display to the user.

Run through the checkout again, and this time it will fail and show the error message:

![Braintree error message](braintree_error_message)

Now we're handling both cases that can happen when processing transactions with Braintree.

## Conclusion

Braintree provides a lot more than just this simple transaction processing. It also handles subscriptions and securely storing user's credentials so that they can re-use them on subsequent checkouts.

I hope you learned something by reading this guide.

* TODO:

* Update app to use Bootstrap
* Re-take screenshots
