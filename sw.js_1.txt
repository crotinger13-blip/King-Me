// King Me — offline asset cache
//
// All the game's card art and piece art lives on GitHub (raw.githubusercontent.com),
// not embedded in index.html, to keep the HTML file a manageable size. That means
// none of it is available offline unless we deliberately cache it ourselves — this
// file is that caching layer. Sound effects (once added) should get appended to
// GAME_ASSET_URLS below the same way, and index.html's own sound-URL list should be
// kept in sync with this one.
//
// Strategy: whenever the game is online, every asset it actually fetches gets saved
// into this cache automatically (see the fetch handler). On top of that, right after
// this service worker installs, it also kicks off a background fetch of the FULL
// asset list below, so a player who's only drawn a handful of cards in their first
// session still ends up with every card's art cached, not just the ones they
// happened to see. That fetch runs in the background and never blocks gameplay —
// if it's still running when the tab closes, whatever it already finished stays
// cached, and the fetch handler will pick up anything it missed the next time that
// asset is actually requested online.
//
// Bump CACHE_NAME (e.g. 'king-me-assets-v2') whenever art is replaced/renamed, so
// stale cached images don't shadow the new ones — the activate handler below
// deletes any cache whose name doesn't match the current version.

const CACHE_NAME = 'king-me-assets-v1';
const ASSET_ORIGIN = 'raw.githubusercontent.com';

const GAME_ASSET_URLS = [
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/Assassinate.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/ace%20up%20the%20sleeve.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/ambush.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/ballista%20fire.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/bear%20trap.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/blizzard.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/blood%20oath.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/bodyguard.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/catapult.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/cavalry-charge.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/chariot%20charge.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/common%20card.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/conscript.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/counter.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/coup%20d'%C3%89tat.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/cross%20strike.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/dead%20man's%20hand.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/demotion.jpg",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/earthquake.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/feint.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/fog%20of%20war.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/hero's%20gambit.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/king%20me.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/last%20stand.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/locust%20swarm.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/mad%20cow.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/main%20menu.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/meteor%20strike.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/once%20more.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/phantom%20march.jpg",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/plague.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/plus%20one.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/puppet%20master.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/rare%20card.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/reinforcements.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/retreat.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/scorched%20earth.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/shield%20wall.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/side%20step.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/siege.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/the%20jester.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/the%20phalanx.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/thor's%20hammer.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/tidal%20wave.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/tornado.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/trojan%20horse.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/uncommon%20card.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/usurp.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/veteran.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/war%20horse.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/war%20tax.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/wildfire.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/cards/wrath.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_1.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_10.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_11.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_12.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_13.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_14.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_15.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_16.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_17.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_18.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_2.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_3.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_4.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_5.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_6.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_7.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_8.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/enemy_9.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_1.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_10.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_11.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_12.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_13.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_14.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_15.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_16.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_17.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_18.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_2.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_3.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_4.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_5.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_6.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_7.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_8.png",
  "https://raw.githubusercontent.com/crotinger13-blip/King-Me/main/assets/pieces/yours_9.png"
];

// Fetch with a hard timeout. Plain fetch() has no built-in timeout — a single
// stalled connection (bad wifi, a captive portal, a proxy that accepts but
// never answers) would otherwise hang forever with neither a resolve nor a
// reject, which is exactly the kind of thing that must never be allowed to
// wedge the bulk warm-up below.
function fetchWithTimeout(url, ms) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), ms);
  return fetch(url, { mode: 'cors', signal: controller.signal }).finally(() => clearTimeout(timer));
}

self.addEventListener('install', (event) => {
  // IMPORTANT: the ~89-asset warm-up below MUST stay wrapped in
  // event.waitUntil(). That's not just about making install "wait" — it's the
  // only signal that tells the browser to keep this service worker alive
  // while the download is still in flight. An earlier version of this file
  // deliberately detached the warm-up from waitUntil to avoid install hanging
  // forever on a stalled request — but that meant the browser had no reason
  // to keep the worker running once the (near-instant) cache.open/skipWaiting
  // step resolved, so the bulk download got cut short after only whatever
  // handful of images happened to finish in that first fraction of a second,
  // even during an otherwise-long play session. The per-request timeout
  // below (fetchWithTimeout) is what makes it safe to wait on this again: the
  // whole batch is now bounded at ~20 seconds worst case (every request times
  // out) rather than being able to hang forever, so there's no longer a
  // reason to keep it detached.
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) =>
      Promise.allSettled(
        GAME_ASSET_URLS.map((url) =>
          fetchWithTimeout(url, 20000).then((res) => {
            if (res && res.ok) return cache.put(url, res);
          }).catch(() => {
            // A single missing/renamed/unreachable/timed-out asset shouldn't
            // stop the other assets from caching successfully.
          })
        )
      ).then(() => self.skipWaiting())
    )
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((names) =>
      Promise.all(
        names
          .filter((name) => name !== CACHE_NAME)
          .map((name) => caches.delete(name))
      )
    ).then(() => self.clients.claim())
  );
});

self.addEventListener('fetch', (event) => {
  const req = event.request;
  // Only ever intervene on GET requests for this game's own art host. Firebase
  // reads/writes, Google Fonts, and everything else pass straight through
  // untouched — this worker has no opinion about them, and must never cache a
  // POST/PUT (e.g. a leaderboard write) or it could replay it later by mistake.
  if (req.method !== 'GET') return;
  let url;
  try { url = new URL(req.url); } catch (err) { return; }
  if (url.hostname !== ASSET_ORIGIN) return;

  event.respondWith(
    caches.match(req).then((cached) => {
      if (cached) return cached;
      return fetch(req).then((res) => {
        if (res && res.ok) {
          const copy = res.clone();
          caches.open(CACHE_NAME).then((cache) => cache.put(req, copy));
        }
        return res;
      }).catch(() => {
        // Truly offline and not already cached — nothing more we can do;
        // let the failed request surface as a normal broken image, same as
        // it would without a service worker at all.
        return cached;
      });
    })
  );
});
