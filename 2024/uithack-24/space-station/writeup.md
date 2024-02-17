# Space Station

The Spaceship Enigma is trying to locate the Stellar Nexus Alpha space station but are having troubles due to treacherous space dust in the area. The ship is receiving strange transmission which they think may be the final piece to locate the space station.

## First thoughts

We get a attachment file with API documentation  
  
Here we get 3 endpoints:  
* GET /get_encrypted
* GET /get_rotors
* POST /post_decrypt

This didn't say much to me except that I need to use data from rotors to decrypt encrypted text.  
When I've decrypted the text I send it into the post_decrypt endpoint as a key.  

  

## Solving
I wasn't quite sure what to do at first, so I searched up rotor ciper.
I then found out this is inspired by Enigma Machine, and the description also included Enigma.

I took the time to understand [How did the Enigma Machine work?](https://www.youtube.com/watch?v=ybkkiGtJmkM)  
During the video they showcased how you take a character and make it go through different rotors that changes the character.  
I thought it would be the same then to get the decrypted key.

## Solvescript
```py
import requests

api_url = "https://uithack.td.org.uit.no:8009"

encrypted_request = requests.get(f"{api_url}/get_encrypted")
encrypted_str = encrypted_request.text


rotors_request = requests.get(f"{api_url}/get_rotors")
rotors_json = rotors_request.json()

def find(s, ch):
    return [i for i, ltr in enumerate(s) if ltr == ch]

stringsArray = []

for strChar in encrypted_str[:-2]:
    rotor1 = rotors_json[0]["rotor1"]
    rotor2 = rotors_json[0]["rotor2"]
    rotor3 = rotors_json[0]["rotor3"]
    stringsArray.append(rotor3[rotor2[rotor1[strChar]]])

obj = {"key": ''.join(stringsArray)}

request = requests.post(f"{api_url}/post_decrypt", json = obj)

print(request.text)
```
