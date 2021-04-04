Hey everyone, 

I want to show everyone what I did to resolve some of the major issues I've been running into during the installation @dupontgu's amazing project. 

Some background: I don't have an iPod or anything, and this was all done on my Raspberry Pi 4 in a PiBoy DMG. My plan for this is to be able to launch spotipy via RetroPie. 

# [Errno 98] Address already in use
After going through the installation I tried running spotipy.py, but I ran into a couple of issues with the [Errno 98] Address in use. I went out of my mind thinking it was related to the UDP_PORT or even the OAuth port. Maybe it was the server address or redis? Nah, throw all of that out the window. This issue is because of a .cache not being found. 

```
python3 spotifypod.py
Traceback (most recent call last):
File "spotifypod.py", line 12, in
from view_model import *
File "/home/pi/retro-ipod-spotify-client/frontend/view_model.py", line 17, in
spotify_manager.refresh_data()
File "/home/pi/retro-ipod-spotify-client/frontend/spotify_manager.py", line 159, in refresh_data
results = sp.current_user_saved_tracks(limit=pageSize, offset=0)
File "/home/pi/.local/lib/python3.7/site-packages/spotipy/client.py", line 1183, in current_user_saved_tracks
return self._get("me/tracks", limit=limit, offset=offset)
File "/home/pi/.local/lib/python3.7/site-packages/spotipy/client.py", line 291, in _get
return self._internal_call("GET", url, payload, kwargs)
File "/home/pi/.local/lib/python3.7/site-packages/spotipy/client.py", line 221, in _internal_call
headers = self._auth_headers()
File "/home/pi/.local/lib/python3.7/site-packages/spotipy/client.py", line 212, in _auth_headers
token = self.auth_manager.get_access_token(as_dict=False)
File "/home/pi/.local/lib/python3.7/site-packages/spotipy/oauth2.py", line 481, in get_access_token
"code": code or self.get_auth_response(),
File "/home/pi/.local/lib/python3.7/site-packages/spotipy/oauth2.py", line 436, in get_auth_response
return self._get_auth_response_local_server(redirect_port)
File "/home/pi/.local/lib/python3.7/site-packages/spotipy/oauth2.py", line 402, in _get_auth_response_local_server
server = start_local_http_server(redirect_port)
File "/home/pi/.local/lib/python3.7/site-packages/spotipy/oauth2.py", line 1300, in start_local_http_server
server = HTTPServer(("127.0.0.1", port), handler)
File "/usr/lib/python3.7/socketserver.py", line 452, in init
self.server_bind()
File "/usr/lib/python3.7/http/server.py", line 137, in server_bind
socketserver.TCPServer.server_bind(self)
File "/usr/lib/python3.7/socketserver.py", line 466, in server_bind
self.socket.bind(self.server_address)
OSError: [Errno 98] Address already in use
```
sebakitzing's post (https://github.com/dupontgu/retro-ipod-spotify-client/issues/22) led me to the right path with this issue.
You can verify if the .cache folder exists by going to /retro-ipod-spotify-client/frontend and typing `ls -a`
I was able to authorize spotipy through Midori browser, but for some reason, the .cache file would not create. I looked around and found Perelin's solution for Spotipy's OAuth (https://github.com/perelin/spotipy_oauth_demo). With a little modification to the spotipy_oauth_demo, I was able to create a cache folder to the /home/pi/ folder:

Please note: you will have to add your own client ID and secret from your Spotify Developer Dashboard. You will also need to add http://localhost:8080 to your Redirect URIs in the dashboard. I will explain the scoping modifications in the next topic.
```
SPOTIPY_CLIENT_ID = 'XXXXXXX'
SPOTIPY_CLIENT_SECRET = 'XXXXXX'
SPOTIPY_REDIRECT_URI = 'http://localhost:8080'

SCOPE = "user-follow-read," \
        "user-library-read," \
        "user-library-modify," \
        "user-modify-playback-state," \
        "user-read-playback-state," \
        "user-read-currently-playing," \
        "app-remote-control," \
        "playlist-modify," \
        "playlist-read-private," \
        "playlist-read-collaborative," \
        "playlist-modify-public," \
        "playlist-modify-private," \
        "streaming," \
        "user-follow-modify," \
        "user-follow-read"


CACHE = '/home/pi/.spotipyoauthcache'

sp_oauth = oauth2.SpotifyOAuth( SPOTIPY_CLIENT_ID, SPOTIPY_CLIENT_SECRET,SPOTIPY_REDIRECT_URI,scope=SCOPE,cache_path=CACHE)

```
With this modification, a cache folder was successfully made! I then 
`sudo cp /home/pi/.spotipyoauthcache /retro-ipod-spotify-client/frontend/.cache`
This will copy the cache folder into the folder to allow spotipypod.py to work. This will also bypass the "Enter the URL you were redirected to:" prompt.
Note: I use cp instead of mv because I wanted to have a backup of the cache folder in case of a disaster.

After copying this cache folder over, I ran into another issue where spotipypod.py cannot write into the cache folder due to permission issues. This can easily be resolved with
`sudo chomod 777 /retro-ipod-spotify-client/frontend/.cache`
This command opens the permission rights to the .cache folder. 

After that, I was able to run spotipypod.py...into another issue.

# Scope Errors
Another issue occurred when running spotipypod.py, somewhere along the line of:
```
https://api.spotify.com/v1/me/following?type=artist&limit=50
spotipy.client.SpotifyException: http status: 403, code:-1 - https://api.spotify.com/v1/me/albums?limit=100&offset=0:
     Insufficient client scope
```
This can be resolved by adding more scopes into spotify_manager.py
```
scope = "user-follow-read," \
        "user-library-read," \
        "user-library-modify," \
        "user-modify-playback-state," \
        "user-read-playback-state," \
        "user-read-currently-playing," \
        "app-remote-control," \
        "playlist-modify," \
        "playlist-read-private," \
        "playlist-read-collaborative," \
        "playlist-modify-public," \
        "playlist-modify-private," \
        "streaming," \
        "user-follow-modify," \
        "user-follow-read"

```

This fixed pretty much my scope issues.

# NoneType Error
This part was tricky. As a person of many many... many playlists, some tracks become unavailable or null. spotipypod.py doesn't know what to do with these null tracks and will end up in an error before even starting. I had to modify some code around spotify_manager.py to skip null tracks. Here are the instances and their code:

**!!Oh yeah before doing this, make sure to fix the playlist order first according to HerrEurobeat's post (https://github.com/dupontgu/retro-ipod-spotify-client/pull/26).**
```
def get_playlist_tracks(id):
    tracks = []
    results = sp.playlist_tracks(id, limit=pageSize)
    while(results['next']):
        for _, item in enumerate(results['items']):
            track = item['track']
            if track is None:
                continue
            else:
                tracks.append(UserTrack(track['name'], track['artists'][0]['nam$
        results = sp.next(results)
    for _, item in enumerate(results['items']):
        if item['track'] is None:
            continue
        else:
            track = item['track']
            tracks.append(UserTrack(track['name'], track['artists'][0]['name'],$
    return tracks

```

```

def refresh_data():
  <--code skip--->
    results = sp.current_user_playlists(limit=pageSize)
    totalindex = 0 # variable to preserve playlist sort index when calling offset loop down below
    while(results['next']):
        offset = results['offset']
        for idx, item in enumerate(results['items']):
            if item['uri'] is None:
                continue
            else:
                tracks = get_playlist_tracks(item['id'])
                DATASTORE.setPlaylist(UserPlaylist(item['name'], totalindex, item['uri'], len(tracks)), tracks, inde$
                totalindex = totalindex + 1
        results = sp.next(results)

```
We're not done yet!

# StartX

I kept running into an issue with the Xauth: `xauth: timeout in locking authority file /home/pi/.Xauthority`. I was not sure what could be causing it, but I read that this was not really needed and safe to deleted. So I did, I ran `sudo rm /home/pi/.Xauthority` I ran into another issue where when using startx, it will not launch and returns an error 
`Cannot Open /dev/tty0 (Permission Denied)` to resolve this, I had to use `sudo chown pi /dev/tty0`. This works, but resets when after rebooting. So I created a script:

```
sudo chown pi /dev/tty0
sudo chown pi /dev/tty1
sudo chown pi /dev/tty2
sudo chown pi /dev/tty3
sudo chown pi /dev/tty4
sudo chown pi /dev/tty5
sudo chown pi /dev/tty6
sudo chown pi /dev/tty7


startx

```
The reason why I repeated this step all the way until tty7 was because of the virtual consoles. If you already have a startx console running, using startx again will create another virtual console, which is a different ttyX set. Setting chown for 8 tty will make it safe to load in any one of these. I know this is a bit of a mess, but I have not run into any issues as of yet. I am open for optimizations and improvements.

make sure to use `sudo chmod +x [yourscript].sh` to allow running ./[yourscript].sh

**!!Please note using `sudo startx` will launch it in root, which will not launch spotipypod.py at all. It'll just give you a blank screen. Just remove `sudo` and it should launch correctly.**

After tackling these issues, I was able to load spotipypod.py at last.

Side note: During this process, I followed HerrEurobeat's other post (https://github.com/dupontgu/retro-ipod-spotify-client/pull/24) on removing emoticons from playlist names so it doesn't error. I have a couple of playlists with this and for 'future-proofing this definitely did the job.

Cheers
