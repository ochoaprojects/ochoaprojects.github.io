---
title: How to use Sony Remote Play on Linux
date: 2022-05-31 12:00:00 -500
categories: [python]
tags: [python,sony,playstation,linux,chiaki,psn]
---

Chiaki is a Free and Open Source Software Client for PlayStation 4 and PlayStation 5 Remote Play for Linux, FreeBSD, OpenBSD, Android, macOS, Windows, Nintendo Switch and potentially even more platforms.

# Installing
You can either download a pre-built release or build Chiaki from source. This download will be for the remote device you want to stream to.

# Usage
If your Console is on your local network, is turned on or in standby mode and does not have Discovery explicitly disabled, Chiaki should find it. Otherwise, you can add it manually. To do so, click the "+" icon in the top right, and enter your Console's IP address.

You will then need to register your Console with Chiaki. You will need two more pieces of information to do this.

# Obtaining your PSN AccountID
Starting with PS4 7.0, it is necessary to use a so-called "AccountID" as opposed to the "Online-ID" for registration (streaming itself did not change). This ID seems to be a unique identifier for a PSN Account and it can be obtained from the PSN after logging in using OAuth. A Python 3 script which does this is provided at [psn-account-id.py](psn-account-id.py) or can be copied from below. Simply run it in a terminal and follow the instructions. Once you know your ID, write it down. You will likely never have to do this process again.

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import sys
if sys.version_info < (3, 0, 0):
	print("DO NOT use Python 2.\nEVER.\nhttps://pythonclock.org")
	exit(1)

import platform
oldexit = exit
def exit(code):
	if platform.system() == "Windows":
		import atexit
		input("Press Enter to exit.")
	oldexit(code)

if sys.stdout.encoding.lower() == "ascii":
	import codecs
	sys.stdout = codecs.getwriter('utf-8')(sys.stdout.buffer)
	sys.stderr = codecs.getwriter('utf-8')(sys.stderr.buffer)

try:
	import requests
except ImportError as e:
	print(e)
	if platform.system() == "Windows":
		from distutils.util import strtobool
		a = input("The requests module is not available. Should we try to install it automatically using pip? [y/n] ")
		while True:
			try:
				a = strtobool(a)
				break
			except ValueError:
				a = input("Please answer with y or n: ")
		if a == 1:
			import subprocess
			subprocess.call([sys.executable, "-m", "pip", "install", "requests"])
		else:
			exit(1)
	else:
		print("\"requests\" is not available. Install it with pip or your distribution's package manager.")
		exit(1)

import requests

from urllib.parse import urlparse, parse_qs, quote, urljoin
import pprint
import base64

# Remote Play Windows Client
CLIENT_ID = "ba495a24-818c-472b-b12d-ff231c1b5745"
CLIENT_SECRET = "mvaiZkRsAsI1IBkY"

LOGIN_URL = "https://auth.api.sonyentertainmentnetwork.com/2.0/oauth/authorize?service_entity=urn:service-entity:psn&response_type=code&client_id={}&redirect_uri=https://remoteplay.dl.playstation.net/remoteplay/redirect&scope=psn:clientapp&request_locale=en_US&ui=pr&service_logo=ps&layout_type=popup&smcid=remoteplay&prompt=always&PlatformPrivacyWs1=minimal&".format(CLIENT_ID)
TOKEN_URL = "https://auth.api.sonyentertainmentnetwork.com/2.0/oauth/token"

print()
print("########################################################")
print("           Script to determine PSN AccountID")
print("                  thanks to grill2010")
print("     (This script will perform network operations.)")
print("########################################################")
print()
print("➡️  Open the following URL in your Browser and log in:")
print()
print(LOGIN_URL)
print()
print("➡️  After logging in, when the page shows \"redirect\", copy the URL from the address bar and paste it here:")
code_url_s = input("> ")
code_url = urlparse(code_url_s)
query = parse_qs(code_url.query)
if "code" not in query or len(query["code"]) == 0 or len(query["code"][0]) == 0:
	print("☠️  URL did not contain code parameter")
	exit(1)
code = query["code"][0]

print("🌏 Requesting OAuth Token") 

api_auth = requests.auth.HTTPBasicAuth(CLIENT_ID, CLIENT_SECRET)
body = "grant_type=authorization_code&code={}&redirect_uri=https://remoteplay.dl.playstation.net/remoteplay/redirect&".format(code)

token_request = requests.post(TOKEN_URL,
	auth = api_auth,
	headers = { "Content-Type": "application/x-www-form-urlencoded" },
	data = body.encode("ascii"))

print("⚠️  WARNING: From this point on, output might contain sensitive information in some cases!")

if token_request.status_code != 200:
	print("☠️  Request failed with code {}:\n{}".format(token_request.status_code, token_request.text))
	exit(1)

token_json = token_request.json()
if "access_token" not in token_json:
	print("☠️  \"access_token\" is missing in response JSON:\n{}".format(token_request.text))
	exit(1)
token = token_json["access_token"]

print("🌏 Requesting Account Info")

account_request = requests.get(TOKEN_URL + "/" + quote(token), auth = api_auth)

if account_request.status_code != 200:
	print("☠️  Request failed with code {}:\n{}".format(account_request.status_code, account_request.text))
	exit(1)

account_info = account_request.json()
print("🥦 Received Account Info:")
pprint.pprint(account_info)

if "user_id" not in account_info:
	print("☠️  \"user_id\" is missing in response or not a string")
	exit(1)

user_id = int(account_info["user_id"])
user_id_base64 = base64.b64encode(user_id.to_bytes(8, "little")).decode()

print()
print("🍙 This is your AccountID:")
print(user_id_base64)
exit(0)
```

1.
![First Prompt of PyScript](/project-assets/SonyRemotePlayOnLinux/FirstPromptOfPyScript.png)
Run the Python Script in a Terminal.

*Note* I'm using Visual Studio Code to run this, so I am able to click the link directly from the terminal to visit the necessary page.

2.
![Sony Sign In](/project-assets/SonyRemotePlayOnLinux/SonySignIn.png)
Sign into your Sony Playstation Account.

3.
![Output from Redirected URL](/project-assets/SonyRemotePlayOnLinux/OutputFromReditedURL.png)
Paste the URL you had after the "Redirect" back into the temrinal with the Python Script. This will output information for the account you signed into, including the "AccountID" that we need for connecting Chiaki.

# Obtaining a Registration PIN
To register a Console with a PIN, it must be put into registration mode. To do this on a PS4, simply go to: Settings -> Remote Play -> Add Device, or on a PS5: Settings -> System -> Remote Play -> Link Device.

You can now double-click your Console in Chiaki's main window to start Remote Play.