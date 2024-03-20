# IMPORTANT NOTICE

This project is now archived as I did not manage to keep up with Google's changes
in their login mechanism.

# yt2sp - YouTube To Spotify

This tool synchronizes your YouTube playlists to Spotify playlists,
intelligently searching for artist & title based on the YouTube video's
name, skipping duplicates and offering the possibility for manual
overrides.

This can be useful if you use YouTube for music discovery, add the music
videos you like to a playlist and want to ensure that your discovered
items are added to Spotify as well which you use for regular music
listening.

In order to work, it requires access to your YouTube and Spotify accounts,
but it runs entirely in your browser and communicates only with YouTube
and Spotify servers and does not store any data at all.

## How to use

As this tool is a web application, you can use the hosted version at 
[https://yt2sp.github.io](https://yt2sp.github.io).

Note that this version uses free API keys therefore might get throttled/limited/banned
by Google or Spotify if used extensively. In this case you are welcome to
run it on your own.

## Building and running

This tool is a web application written in [Svelte](https://svelte.dev), which compiles to
a bunch of static HTML/JS/CSS files.

To compile, first you will need to supply the API keys that will be embedded in the generated
files. You can specify them as environment variables or in a `.env` file:

- `GOOGLE_CLIENTID`: Oauth 2.0 Client ID for Google APIs, generated on the [Google Developers Console](https://console.developers.google.com/apis/credentials)
- `GOOGLE_APIKEY`: API key for YouTube Data API v3 generated on the [Google Developers Console](https://console.developers.google.com/apis/credentials)
- `SPOTIFY_CLIENTID`: OAuth 2.0 Client ID for the Spotify API, generated on the [Spotify Developers Console](https://developer.spotify.com/dashboard/applications).

Make sure that the credentials are configured with the proper origin.

With these, you can compile the source with the following commands (you will
need [Node.js and npm](https://nodejs.org/) installed):

    $ git clone https://github.com/yt2sp/yt2sp.git
    $ cd yt2sp
    $ npm install
    $ npm run build

The compiled files will be in the `public` directory, which you can serve with any kind of static
web hosting, or alternatively, directly from the directory with the following command:

    $ npm start

## License

This tool is licensed under GPL 3.0.