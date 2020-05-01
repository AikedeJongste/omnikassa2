# Omnikassa2

This Gem provides the Ruby on Rails integration for the new Omnikassa 2.0 JSON API from the
Rabobank. The documentation for this API is currently here:
[Rabobank.nl](https://www.rabobank.nl/images/handleiding-merchant-shop_29920545.pdf)


## Installation

Add this line to your application's Gemfile:

```ruby
gem 'omnikassa2', git: 'https://github.com/AikedeJongste/omnikassa2.git', branch: 'master'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install omnikassa2


## Usage

### Set environment variables

This gem reads it's config and tokens from environment variables. You have to
set at least the following:

* OMNIKASSA_REFRESH_TOKEN
* OMNIKASSA_SIGNING_KEY
* OMNIKASSA_CURRENCY
* OMNIKASSA_RETURN_URL

The merchant return url is the page in your application where the user lands
after completing the transaction on the Rabobank.nl site.

### Create initializer

Create an initializer in config/initializers/omnikassa.rb to read the
environment variables.

```ruby
if ENV['OMNIKASSA_REFRESH_TOKEN'].present?
  Omnikassa2.config(
    refresh_token:       ENV['OMNIKASSA_REFRESH_TOKEN'],
    signing_key:         ENV['OMNIKASSA_SIGNING_KEY'],
    currency:            ENV['OMNIKASSA_CURRENCY'],
    environment:         ENV['RAILS_ENV'],
    merchant_return_url: ENV['OMNIKASSA_RETURN_URL']
  )
end
````

### Create a route

You need a route to process the incoming Notify post from the Rabobank. They
call it Webhook-url in the Rabobank Dashboard. Set it to something like
yourapp.com/omnikassa and add this route to your routes file:

```ruby
    post 'omnikassa' => 'payments#omnikassa_notify'
```

### Example

1. Refresh access token when needed.

```ruby
  # Obtain new token when no valid token is available in local storage (e.g. db, disk, cache).
  token_data = JSON.parse(Omnikassa2::Refresh.connect)
  access_token = token_data.fetch('token')
  expires_at = token_data.fetch('validUntil')

  # Store these values locally;
  # Only obtain a new token when the stored token has (almost) expired.
```

2. Initialize order (start payment).

```ruby
  # First, obtain access token (see 1)
  # access_token = ...

  # Use access token
  Omnikassa2.set_access_token(access_token)

  # Announce payment
  announce = Omnikassa2::Announce.new(
    amount: '100',                # amount in cents
    merchant_order_id: '...',     # own payment ID
    payment_brand: 'IDEAL'        # IDEAL | BANCONTACT | ...
  )

  # Redirect user to Omnikassa payment URL
  redirect_to announce.redirect_url
```

3. Handle payment return (user is redirected by OmniKassa after payment).

```ruby
  # This code could be in a Rails controller action.

  # GET
  def new
    # `params` contains OmniKassa data
    payment_return = Omnikassa2::PaymentReturn.new(params)
    order_data = payment_return.verified_data

    # Example of handling order status details:

    # Find local order using own payment ID (see 2)
    order = Order.find_by_hashid(order_data['order_id'])

    # Update local order
    if order.present?
      order.response_data = order_data.to_json # optionally store all details with order
      order.payment_result = order_data['status'] # e.g. 'COMPLETED', 'IN_PROGRESS', 'EXPIRED'
                                                  # or 'CANCELLED'
      order.save # or wait for async update, see 4.
    end

    # render or redirect, display payment result to user
    # ...
  end
```

4. Process async payment status update by Omnikassa).

```ruby
  # This code could be in a Rails controller action.
  # The corresponding route should be accessible by Omnikassa.

  # POST
  def omnikassa_notify
    # `params` contains Omnikassa data
    notification = Omnikassa2::Notification.new(params)
    data = notification.verified_data
    status = Omnikassa2::Status.new(data['authentication'])

    # Example of handling order status details (possibly multiple orders):
    status.order_results.each do |order_data|
      # Find local order using own payment ID (see 2)
      order = Order.find_by_hashid(order_data['merchantOrderId'])
      next if order.nil?

      # Update local order
      order.response_data = order_data.to_json # optionally store all details with order
      order.payment_result = order_data['orderStatus'] # e.g. 'COMPLETED'
      order.save
    end

    # Return status 200
    head :ok
  end
```

## Development

Feel free to contact us if you need help implementing this Gem in your
application. Also let us know if you need additional features.

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/[USERNAME]/omnikassa2.
