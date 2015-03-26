

# Introduction #

This module provides Python programmers an easy interface to the [TextMagic HTTPS API](http://api.textmagic.com/https-api).

# TextMagic HTTPS API #

As explained at http://api.textmagic.com/https-api/textmagic-api-commands, the TextMagic API has these commands:
  * send
  * account
  * message\_status
  * receive
  * delete\_reply
  * check\_number

It is also possible to set up a "Callback URL" to be notified of:
  * delivery status of sent messages
  * received messages

# Python API #

To use the API, you need to instantiate a "client" first:
```
import textmagic.client
client = textmagic.client.TextMagicClient('your_username', 'your_api_password')
```

All commands are implemented as methods on the `TextMagicClient` class.

## Commands ##

### send ###

The `send` command allows sending of one or more messages.

It takes these parameters:
  * `text`: Message text. Unicode characters are allowed, but using Unicode affects the maximum message length.
  * `phone`: A single phone number or a list of phone numbers. A number consists of digits only - starting with the country code.
  * `max_length`: (Optional) Maximum number of messages to split text into. By default a message is split automatically into a maximum of three separate messages. The maximum length for an ASCII message is around 160 characters and for Unicode it is around 70 characters. You can read more about message length at http://api.textmagic.com/https-api/message-length.
  * `send_time`: (Optional) A `time.struct_time` or a "unix time" (seconds since the epoch) indicating when the message is to be sent.

The response contains:
  * `message_id`: A dictionary with system allocated message ids as keys and destination phone numbers as values.
  * `sent_text`: Speaks for itself.
  * `parts_count`: Number of messages the text was split into.

For example:
```
result = client.send("Hello, World!", "1234657890")
message_id = result['message_id'].keys()[0]
```
or:
```
result = client.send("Hello, Worlds!", ["1234657890", "2345678901"])
message_ids = result['message_id'].keys()
```
also:
```
result = client.send("Hello, Future!", "1234657890", send_time=time.strptime("2009-07-07 13:55:45", "%Y-%m-%d %H:%M:%S"))
```
or:
```
result = client.send("Hello, Future!", "1234657890", send_time=time.time()+300)
```


### account ###

The `account` command retrieves your account balance.

It takes no parameters.

The response contains:
  * `balance`: The number of credits you have left in your account.

For example:
```
balance = client.account()['balance']
```

### message\_status ###

The `message_status` command reports on the delivery status of sent messages.

It takes one parameter:
  * `ids`: One message id, or a list of message ids.

The response is a dictionary keyed on the message ids, with each value containing:
  * `text`: The text of the message.
  * `status`: Delivery codes as explained at http://api.textmagic.com/https-api/sms-delivery-notification-codes.
  * `created_time`: Time the message was created as a `time.struct_time` (local time).
  * `reply_number`: Phone number to which replies can be sent.
  * `credits_cost`: (Optional) Number of credits used.
  * `completed_time`: (Optional) Time the message lifecycle was completed as a `time.struct_time` (local time).

For example:
```
response = client.message_status(message_id)
message_info = response[message_id]
delivery_status = message_info['status']
time_string = time.strftime("%a, %d %b %Y %H:%M:%S", message_info['created_time'])
```

### receive ###

The `receive` command receives messages from your Inbox.

It takes one parameter:

  * `last_retrieved_id`: Id of last message retrieved from your Inbox.

The response is a dictionary with items:
  * `messages`: A list of dictionaries each containing the following information about one received message:
    * `message_id`: System allocated id for the message.
    * `from`: Number the message was sent from (only digits, starting with the country code).
    * `timestamp`: Received time as a `time.struct_time` (local time).
    * `text`: Message text.
  * `unread`: The number of messages in the Inbox that is still unread.

For example:
```
received_messages = client.receive(0)
messages_info = received_messages['messages']
num_messages = len(messages_info)
for message_info in messages_info:
  print "%(text)s received from %(from)s" % message_info
```

### delete\_reply ###

The `delete_reply` command deletes messages from your Inbox.

It takes one parameter:
  * `ids`: One message id, or a list of message ids.

The response is a dictionary with one item:
  * `deleted`: A list of deleted message ids

For example:
```
client.delete_reply(message_info['message_id'])
client.delete_reply(['123','456'])
```

### check\_number ###

The `check_number` command provides country and pricing information for a phone number.

It takes one parameter:
  * `phone`: One message id, or a list of message ids.

The response is a dictionary keyed on the phone numbers, with each value containing:
  * `price`: Number of credits used when sending to this country.
  * `country`: Country of the phone number. Here is a complete list of [country prefixes](https://www.textmagic.com/app/wt/messages/new/cmd/get_countries).

For example:
```
response = client.check_number('44123456789')
info = response['44123456789']
country = info['country']
credits = info['price']
```
or:
```
response = client.check_number(['44123456789', '27123456789'])
```

## Callback URLs ##

The TextMagic API website explains how to set up [Callback URLs](http://api.textmagic.com/https-api/receiving-delivery-notifications-and-incoming-sms-messages-via-callback-urls). A Callback URL is used by the TextMagic system to send notifications of:
  * delivery status of sent messages and
  * incoming messages

Notification is implemented as a POST request to the defined callback URL.

### callback\_message ###

When a notification is received you can call `callback_message`, passing it the POST parameters as a dictionary. It responds with either a `StatusCallbackResponse` or a `ReplyCallbackResponse`.

#### StatusCallbackResponse ####
This is a notification of the success or failure of message delivery.

It is a dictionary with keys:
  * `status`: Delivery status; see http://api.textmagic.com/https-api/sms-delivery-notification-codes.
  * `message_id`: Message id
  * `timestamp`: Timestamp as a `time.struct_time` (local time).
  * `credits_cost`: Amount of credits used to send this message.

#### ReplyCallbackResponse ####
This is a notification of an incoming SMS.

It is a dictionary with keys:
  * `message_id`: Message id
  * `text`: Text of the message.
  * `timestamp`: Timestamp as a `time.struct_time` (local time).
  * `from`: Telephone number the message was sent from.

## Errors ##

A `TextMagicError` is raised when an error response is received from the TextMagic server. The list of error codes and messages is available at http://api.textmagic.com/https-api/api-error-codes.