<script>
  const REDIRECT = window.location.href.slice(0, -1)
  const SPOTIFY_STATE = 'yt2sp'
  const GOOGLE_SCOPE = 'https://www.googleapis.com/auth/youtube.readonly'
  const SPOTIFY_SCOPE =
    'user-read-private user-read-email playlist-read-private playlist-modify-private'
  var popup

  var spotifyUser
  var isGoogleAuthorized
  var youtubePlaylists
  $: spotifyUserId = spotifyUser && spotifyUser.id
  $: spotifyPlaylists = loadSpotifyPlaylists(spotifyUserId)
  var selectedYoutube
  var selectedSpotify

  $: syncPossible =
    selectedYoutube && selectedSpotify && spotifyUserId && isGoogleAuthorized

  var youtubeItems = []
  var spotifyItems = []
  var syncTable = []
  var syncDone = []
  var syncCompleted
  var checkboxState = false

  var spotifyToken = localStorage.getItem('spotify_token')

  window.spotifyCallback = (payload) => {
    popup.close()
    spotifyToken = payload
    localStorage.setItem('spotify_token', spotifyToken || '')
    verifySpotify()
  }

  function clearSpotifyAuth() {
    spotifyToken = undefined
    spotifyUser = undefined
    localStorage.removeItem('spotify_token')
  }

  var url = new URL(window.location.href.replace('#', '?'))
  if (url.searchParams.get('state') == SPOTIFY_STATE) {
    window.opener.spotifyCallback(url.searchParams.get('access_token'))
  }

  function spotifyRequest(url, payload) {
    var h = { Authorization: `Bearer ${spotifyToken}` }
    if (payload)
      return fetch(url, {
        headers: h,
        method: 'POST',
        body: JSON.stringify(payload),
      })
    else return fetch(url, { headers: h })
  }

  async function verifySpotify() {
    var r = await spotifyRequest('https://api.spotify.com/v1/me')
    if (r.ok) {
      spotifyUser = await r.json()
    } else {
      clearSpotifyAuth()
    }
  }

  if (spotifyToken) {
    verifySpotify()
  }

  function authSpotify() {
    var url = `https://accounts.spotify.com/authorize?client_id=${SPOTIFY_CLIENTID}&redirect_uri=${REDIRECT}&scope=${SPOTIFY_SCOPE}&response_type=token&state=${SPOTIFY_STATE}`
    if (spotifyUser) {
      url += '&show_dialog=true'
    }
    popup = window.open(url, 'Login with Spotify', 'width=800,height=600')
  }
  let tokenClient

  function gapiInit() {
    gapi.client.init({}).then(function () {
      gapi.client.load(
        'https://www.googleapis.com/discovery/v1/apis/youtube/v3/rest'
      )
    })
  }

  function gapiLoad() {
    gapi.load('client', gapiInit)
  }

  function gisInit() {
    tokenClient = google.accounts.oauth2.initTokenClient({
      client_id: GOOGLE_CLIENTID,
      scope: GOOGLE_SCOPE,
      callback: '',
    })
  }

  window.handleCredentialResponse = (response) => {
    isGoogleAuthorized = true
    tokenClient.callback = async (resp) => {
      if (resp.error !== undefined) {
        throw resp
      }

      var page
      youtubePlaylists = []
      do {
        var r = await gapi.client.youtube.playlists.list({
          part: ['id,snippet,contentDetails'],
          maxResults: 50,
          mine: true,
          pageToken: page,
        })
        youtubePlaylists = youtubePlaylists.concat(r.result.items)
        page = r.result.nextPageToken || null
      } while (page)
    }
    if (gapi.client.getToken() === null) {
      tokenClient.requestAccessToken({ prompt: 'consent' })
    } else {
      tokenClient.requestAccessToken({ prompt: '' })
    }
  }

  async function loadSpotifyPlaylists(user) {
    if (!user) return []
    var page
    var spotifyPlaylists = []
    do {
      var r = await spotifyRequest(
        page || 'https://api.spotify.com/v1/me/playlists?limit=50'
      ).then((r) => r.json())
      spotifyPlaylists = spotifyPlaylists.concat(r.items)
      page = r.next
    } while (page)
    return spotifyPlaylists
  }

  function formatSpotifyItem(i) {
    if (i)
      return (
        i.artists.map((artist) => artist.name).join(', ') +
        '-' +
        i.name +
        ' (' +
        i.album.name +
        ', ' +
        i.album.release_date +
        ')'
      )
    else return '...'
  }

  class SyncItem {
    constructor(source) {
      this.source = source
      this.query = this.source.snippet.title
      this.candidates = []
      this.target = null
      this.selected = false
    }
  }

  async function findCandidates(query) {
    var r = await spotifyRequest(
      `https://api.spotify.com/v1/search?q=${query}&type=artist,track`
    )
    r = await r.json()
    return r.tracks.items
  }

  async function loadYoutubeItems() {
    var page
    youtubeItems = []
    do {
      var r = await gapi.client.youtube.playlistItems.list({
        part: ['snippet,contentDetails'],
        maxResults: 50,
        playlistId: selectedYoutube.id,
        pageToken: page,
      })
      youtubeItems = youtubeItems.concat(r.result.items)
      page = r.result.nextPageToken || null
    } while (page)
  }

  async function loadSpotifyItems() {
    var page
    spotifyItems = []
    do {
      var r = await spotifyRequest(
        page || selectedSpotify.tracks.href + '?limit=50'
      ).then((r) => r.json())
      spotifyItems = spotifyItems.concat(r.items)
      page = r.next
    } while (page)
  }

  function isDuplicate(item) {
    if (item && spotifyItems.length)
      return spotifyItems.find((i) => i.track.id == item.id)
    else return false
  }

  async function sync() {
    await loadYoutubeItems()
    await loadSpotifyItems()

    syncTable = []
    youtubeItems.forEach((youtubeItem) => {
      syncTable = [...syncTable, new SyncItem(youtubeItem)]
    })

    syncTable.forEach(async (item, index) => {
      item.candidates = await findCandidates(item.query)

      if (!item.candidates.length) {
        item.query = item.query
          .replace(/ *[\[\(\{][^)]*[\)\]\}] */g, '')
          .replace(/ *(HD|HQ|720p|1080p|4k) */g, '')
          .replace(/ *(of+icial *)?(music *)?video/g, '')
        item.candidates = await findCandidates(item.query)
      }

      if (!item.candidates.length) {
        item.query = item.query
          .replace(/lyrics?/gi, '')
          .replace(/(feat|ft|featuring)\..*$/gi, '')
          .replace(/with.*$/gi, '')
        item.candidates = await findCandidates(item.query)
      }

      if (!item.candidates.length) {
        item.query = item.query.split('-')[1] || item.query.split('-')[0]
        item.candidates = await findCandidates(item.query)
      }

      item.target = item.candidates[0]
      item.selected = item.target && !isDuplicate(item.target)
      syncTable[index] = item
    })
  }

  function completeSync() {
    syncDone = syncTable.filter((i) => i.selected)
    if (syncDone.length)
      syncCompleted = spotifyRequest(selectedSpotify.tracks.href, {
        uris: syncDone.map((i) => i.target.uri),
      })
  }

  $: syncLength = syncTable
    .filter((i) => i.selected)
    .filter((i) => !isDuplicate(i.target)).length
</script>

<style>
  td {
    vertical-align: middle;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  th {
    background: lightgrey;
  }
  select {
    width: 300px;
    overflow: hidden;
    padding: 2px;
  }
  tr {
    height: 40px;
  }
  input {
    padding: 2px;
  }
  .sync tr:nth-child(odd) {
    background: #eee;
  }
  table {
    border-collapse: collapse;
  }
</style>

<svelte:head>
  <script
    async
    defer
    src="https://apis.google.com/js/api.js"
    on:load={gapiLoad}>

  </script>
  <script
    async
    defer
    src="https://accounts.google.com/gsi/client"
    on:load={gisInit}>

  </script>
</svelte:head>
<main>

  {#if !syncTable.length}
    <h1>YouTube To Spotify</h1>
    <p>
      <a href="https://github.com/yt2sp/yt2sp">Github page</a>
    </p>
    <p>
      This tool synchronizes your YouTube playlists to Spotify playlists,
      intelligently searching for artist & title based on the YouTube video's
      name, skipping duplicates and offering the possibility for manual
      overrides.
    </p>

    <p>
      This can be useful if you use YouTube for music discovery, add the music
      videos you like to a playlist and want to ensure that your discovered
      items are added to Spotify as well which you use for regular music
      listening.
    </p>

    <p>
      In order to work, it requires access to your YouTube and Spotify accounts,
      but it runs entirely in your browser and communicates only with YouTube
      and Spotify servers and does not store any data at all.
    </p>

    <h2>Step 1: Connect accounts:</h2>

    <b>Google:</b>
    <p>

      {#if isGoogleAuthorized}
        Signed in - reload to sign in again
      {:else}
        <div
          id="g_id_onload"
          data-client_id="985321036628-tlp6avu4gnudpr6dpms504lstmoj9uqm.apps.googleusercontent.com"
          data-context="signin"
          data-ux_mode="popup"
          data-callback="handleCredentialResponse"
          data-auto_prompt="false" />

        <div
          class="g_id_signin"
          data-type="standard"
          data-shape="rectangular"
          data-theme="outline"
          data-text="signin_with"
          data-size="large"
          data-logo_alignment="left" />
      {/if}
    </p>

    <p>
      <b>Spotify:</b>
      {#if spotifyUser}Signed in as {spotifyUser.display_name}{/if}
      <button on:click={authSpotify}>
        {#if spotifyUser}Sign out{:else}Sign in{/if}
      </button>
    </p>

    <h2>Step 2: Select source and target playlists:</h2>

    <table>
      <tr>
        <td>
          <b>Source YouTube playlist:</b>
        </td>
        <td>

          {#if youtubePlaylists}
            {#if youtubePlaylists.length}
              <select bind:value={selectedYoutube}>
                {#each youtubePlaylists as playlist}
                  <option value={playlist}>
                    {playlist.snippet.title} ({playlist.contentDetails.itemCount}
                    items)
                  </option>
                {/each}
              </select>
            {:else}...{/if}
          {/if}

        </td>
      </tr>
      <tr>
        <td>
          <b>Target Spotify playlist:</b>
        </td>
        <td>
          {#await spotifyPlaylists}
            ...
          {:then spotifyPlaylists}
            {#if spotifyPlaylists.length}
              <select bind:value={selectedSpotify}>
                {#each spotifyPlaylists as playlist}
                  <option value={playlist}>
                    {playlist.name} ({playlist.tracks.total} items)
                  </option>
                {/each}
              </select>
            {:else}...{/if}
          {/await}

        </td>
      </tr>
    </table>

    <h2>Step 3: Sync!</h2>

    <button on:click={sync} disabled={!syncPossible}>
      Search for items to be synchronized
    </button>
  {:else if !syncDone.length}
    <h2>Review sync actions</h2>
    <button on:click={completeSync} disabled={syncLength === 0}>
      Sync {syncLength} items
    </button>
    &nbsp;
    <button on:click={() => (syncTable = [])}>Cancel sync</button>

    <table class="sync" style="width: 100%; table-layout: fixed">
      <tr>
        <th width="33%">Source video</th>
        <th width="33%">Search term</th>
        <th width="40px" />
        <th width="305px">Potential targets</th>
        <th width="33%">Selected target</th>
        <th width="30px">
          <input
            type="checkbox"
            title="Select all"
            bind:checked={checkboxState}
            on:click={() => {
              syncTable = syncTable.map((i) => {
                i.selected = !checkboxState && !isDuplicate(i.target)
                return i
              })
            }} />
        </th>
      </tr>
      {#each syncTable as item}
        <tr>
          <td>
            <a
              href={'https://www.youtube.com/watch?v=' + item.source.contentDetails.videoId}
              title={item.source.snippet.title}>
              {item.source.snippet.title}
            </a>
          </td>
          <td>
            <input type="text" style="width: 100%" bind:value={item.query} />
          </td>
          <td>
            <button
              on:click={async () => {
                item.candidates = await findCandidates(item.query)
                item.target = item.candidates[0]
                item.selected = item.target && !isDuplicate(item.target)
              }}>
              🔎
            </button>
          </td>
          <td>
            {#await item.candidates}
              ...
            {:then candidates}
              {#if candidates.length}
                <!-- svelte-ignore a11y-no-onchange -->
                <select
                  bind:value={item.target}
                  on:change={() => {
                    item.selected = !isDuplicate(item.target)
                  }}>
                  {#each candidates as candidate}
                    <option value={candidate}>
                      {formatSpotifyItem(candidate)}
                    </option>
                  {/each}
                </select>
              {:else}-{/if}
            {/await}
          </td>
          <td>
            {#await item.target}
              ...
            {:then i}
              {#if i}
                <a href={i.external_urls.spotify} title={formatSpotifyItem(i)}>
                  {formatSpotifyItem(i)}
                </a>
              {/if}
            {/await}
          </td>
          <td style="text-align: center">

            <input
              type="checkbox"
              bind:checked={item.selected}
              disabled={isDuplicate(item.target)} />

          </td>
        </tr>
      {/each}
    </table>
  {:else}
    <h2>Completing sync</h2>
    Syncing {syncDone.length} items:
    <ol>
      {#each syncDone as i}
        <li>{i.source.snippet.title} → {formatSpotifyItem(i.target)}</li>
      {/each}
    </ol>
    {#await syncCompleted}
      <button disabled>...</button>
    {:then i}
      <button
        on:click={() => {
          syncTable = []
          syncDone = []
          spotifyPlaylists = loadSpotifyPlaylists(spotifyUserId)
        }}>
        OK
      </button>
    {/await}
  {/if}

</main>
