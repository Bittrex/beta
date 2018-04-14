#!/usr/bin/python
# -*- coding: utf-8 -*-

# Stanislav Lazarov

# A practical example showing how to connect to the public and private channels of Bittrex.

# If you want to test the private channels, you will have to uncomment the
# Private methods block in the main body and define the API_KEY and API_SECRET.


# Requires Python3.5+
# pip install git+https://github.com/slazarov/python-signalr-client.git

from signalr_aio import Connection
from base64 import b64decode
from zlib import decompress, MAX_WBITS
import hashlib
import hmac
import json

global API_KEY, API_SECRET


async def process_message(message):
    try:
        deflated_msg = decompress(b64decode(message, validate=True), -MAX_WBITS)
    except SyntaxError:
        deflated_msg = decompress(b64decode(message, validate=True))
    return json.loads(deflated_msg.decode())


async def create_signature(api_secret, challenge):
    api_sign = hmac.new(api_secret.encode(), challenge.encode(), hashlib.sha512).hexdigest()
    return api_sign


# Create debug message handler.
async def on_debug(**msg):
    # In case of `queryExchangeState` or `GetAuthContext`
    if 'R' in msg and type(msg['R']) is not bool:
        # For the simplicity of the example I(2) corresponds to `queryExchangeState` and I(3) to `GetAuthContext`
        # Check the main body for more info.
        if msg['I'] == str(2):
            decoded_msg = await process_message(msg['R'])
            print(decoded_msg)
        elif msg['I'] == str(3):
            signature = await create_signature(API_SECRET, msg['R'])
            hub.server.invoke('Authenticate', API_KEY, signature)


# Create error handler
async def on_error(msg):
    print(msg)


# Create hub message handler
async def on_message(msg):
    decoded_msg = await process_message(msg[0])
    print(decoded_msg)


async def on_private(msg):
    decoded_msg = await process_message(msg[0])
    print(decoded_msg)


if __name__ == "__main__":
    # Create connection
    # Users can optionally pass a session object to the client, e.g a cfscrape session to bypass cloudflare.
    connection = Connection('https://beta.bittrex.com/signalr', session=None)

    # Register hub
    hub = connection.register_hub('c2')

    # Assign debug message handler. It streams unfiltered data, uncomment it to test.
    connection.received += on_debug

    # Assign error handler
    connection.error += on_error

    # Assign hub message handler
    # Public callbacks
    hub.client.on('uE', on_message)
    hub.client.on('uS', on_message)
    # Private callbacks
    hub.client.on('uB', on_private)
    hub.client.on('uO', on_private)

    # Send a message
    hub.server.invoke('SubscribeToExchangeDeltas', 'BTC-ETH')  # Invoke 0
    hub.server.invoke('SubscribeToSummaryDeltas')  # Invoke 1
    hub.server.invoke('queryExchangeState', 'BTC-NEO')  # Invoke 2

    # Private methods
    # API_KEY, API_SECRET = '### API KEY ###', '### API SECRET ###'
    # hub.server.invoke('GetAuthContext', API_KEY) # Invoke 3

    # Start the client
    connection.start()
