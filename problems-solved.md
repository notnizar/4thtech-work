# Problems I solved

[← Back to the overview](README.md)

Each entry gives what went wrong, why, and what I did about it. They are grouped by where
the problem was.

- [Security](#security)
- [Devices and firmware](#devices-and-firmware)
- [The phone app and the app stores](#the-phone-app-and-the-app-stores)
- [The IoT server](#the-iot-server)
- [The website](#the-website)
- [Deployment and operations](#deployment-and-operations)

---

## Security

### The MQTT broker was open to anyone

**What went wrong.** A code review of the first IoT server showed that the broker accepted
anonymous connections. Anyone who found the address could read every device's messages and
send commands.

**Why.** The server had startup code to configure the broker's login rules. Against the
broker version in use, one setting was out of the accepted range, so the configuration was
rejected. The code logged a warning and carried on, and the broker stayed open.

**What I did.**
- Devices now connect with mutual TLS: each has its own certificate, and the broker refuses
  a connection without one.
- Anything not explicitly allowed is denied, and each device may use only its own topics.
- The setup code retries, then verifies that the broker really is locked. If it isn't, it
  logs a critical error instead of a warning. It checks again every five minutes.
- Sign-in, sign-up and token refresh got rate limits, and the public internet lost access
  to the metrics endpoint.

### Private keys were being sent to devices

**What went wrong.** In the first design the server created each device's key and sent it
to the device during setup. A key that travels can be copied.

**What I did.** The device now creates its own key on first start and keeps it. It sends
only a signing request; the server certifies the key without ever seeing it.

### Two certificate chains that must never be mixed

**The problem.** The phone app needs a certificate that phones trust, so the API uses Let's
Encrypt. Devices are different: an ESP32 in someone's home is given one certificate
authority at setup and has no way to be told about a new one. If the device-facing
certificate were ever replaced with a Let's Encrypt one, every installed device would stop
connecting and stop being able to update, with no way to recover them remotely.

**What I did.** Two separate chains, on purpose: Let's Encrypt for phones, and a private CA
for devices and firmware downloads. The deployment guide explains which is which and carries
a warning at the exact step where the mistake would be made.

### One visitor could lock everyone out

**What went wrong.** On the website, the sixth contact message per minute, from anyone, was
refused. Worse, a stranger could lock staff out of the admin panel by failing five sign-ins.

**Why.** The rate limiter counted requests per address, and behind the proxy every request
arrived from the proxy's address. All visitors shared one bucket.

**What I did.** Limits now use the visitor's real address, from a header that nginx
overwrites on every request (so a visitor can't forge it). The code no longer trusts the
forwarded-for header, which the client controls.

### A port that skipped the firewall

**What went wrong.** The website's Docker configuration published its internal proxy on
every network interface. Docker-published ports bypass the server's firewall, so the admin
panel and the API would have been reachable over plain HTTP by anyone who found the
server's address.

**What I did.** The port is published on the loopback interface only. The front proxy
reaches it over Docker's internal network.

### The staff panel shared a door with the public site

**What went wrong.** The admin panel was a single 913-line file served from the public
domain by the API.

**What I did.** Split it into its own container on its own hostname. On the public hostname
the panel, the staff API, the sign-in endpoints and the API documentation all answer 404.
Staff requests stay same-origin, so the session cookie keeps its strict settings. Later
I added a required custom header on every staff change (protection against cross-site
requests) and a Content Security Policy.

---

## Devices and firmware

### An unplugged device stayed "online" for two minutes

**What went wrong.** Pull the plug on a device and the app kept showing it as online for a
minute and a half to two minutes. Commands sent to it timed out.

**Why.** The device promised the broker a sign of life every 60 seconds, and the broker
checked only every 30.

**What I did.** A 15-second keepalive in the firmware and a 5-second check on the broker. A
device that loses power now shows as offline within about 30 seconds.

### The ESP32 ran out of memory for TLS

**What went wrong.** With Bluetooth and Wi-Fi both running, the secure connection to the
broker failed with an out-of-memory error.

**Why.** The default Bluetooth stack on a classic ESP32 uses so much memory that not enough
is left for a TLS handshake.

**What I did.** Replaced the Bluetooth stack with NimBLE, a much smaller one, and changed
the partition layout so that Bluetooth, Wi-Fi and TLS fit in flash together. TLS errors now
log the error code and the free memory, so the next such problem is quick to diagnose.

### Setup failed instantly, because of time arithmetic

**What went wrong.** Wi-Fi setup failed the moment it started, and Bluetooth guests were
dropped at once.

**Why.** The main loop read the clock once at the top and compared it with timestamps set
later in the same pass. Those timestamps were newer than the "current" time, so the
unsigned subtraction wrapped around to a huge number, and every time-out looked as if it
had expired long ago.

**What I did.** One helper that computes "milliseconds since" from a fresh clock reading,
used for every time-out.

### Devices showed as offline while they were connected

**What went wrong.** In the first version, some devices never appeared online and their
controls stayed blocked.

**Why.** Device IDs were built from the hardware address, and different parts of the system
wrote it differently (with or without separators, upper or lower case), so the "online"
message never matched the device.

**What I did.** One ID format everywhere: the factory-programmed address, 12 uppercase hex
digits, read from the same source on every board.

### Bluetooth on Android kept disconnecting

**What went wrong.** During setup, Android phones dropped the Bluetooth connection with the
well-known "status 133" error, and scans sometimes found nothing.

**What I did.** Sequenced the handshake so that only one Bluetooth operation runs at a
time, stabilized the packet-size negotiation, asked for the Bluetooth and location
permissions before scanning, and gave the Bluetooth adapter a moment to settle before the
scan starts. Connection loss in the middle of setup is now handled instead of hanging.

### Firmware signing broke on Windows

**What went wrong.** The script that publishes firmware compared the signing key with the
one built into the firmware, and the comparison failed on Windows although the keys were
the same.

**Why.** OpenSSL on Windows writes carriage returns into its output. The key didn't match
byte for byte, and the published signature carried a stray line ending.

**What I did.** The script strips them from both.

### Safe firmware updates

**The problem.** A bad update sent over the air can turn every device into a brick, and a
stolen server could push hostile firmware.

**What I did.** Devices accept only firmware signed with a key that never touches the
server. A device checks the file's hash and signature, installs it beside the running
version, and restarts. If the new version does not reach the cloud within 10 minutes, the
device goes back to the previous one.

---

## The phone app and the app stores

### iPhones could not connect at all

**What went wrong.** The app worked on Android and failed on every iPhone, against the same
server.

**Why.** iOS rejects server certificates that are valid for more than 825 days, even from a
trusted authority. The server's certificate was valid for five years.

**What I did.** A renewal script that reissues the certificate for 820 days with the same
key and the same names, so devices in the field notice nothing, then reloads the proxy and
the broker.

### Sign in with Apple always failed on test phones

**What went wrong.** The Apple button only ever said "Couldn't sign in with Apple."

**Why.** Test builds were being installed with a sideloading tool, which re-signs the app
with a personal Apple ID. Re-signing drops the Sign in with Apple entitlement.

**What I did.** The CI pipeline now uploads every build to App Store Connect, and testers
install from TestFlight with Apple's signature intact.

### A reinstalled app came back signed in

**What went wrong.** Delete the app from an iPhone, install it again, and it opened signed
in, with the old device keys.

**Why.** The iPhone Keychain outlives the app that wrote to it.

**What I did.** The app marks its first run. Empty preferences mean a new install, and the
app then clears what the old one left in the Keychain. An update from an older build keeps
its sign-in.

### "Wrong password" looked like "session expired"

**What went wrong.** Deleting an account asks for the password first. With a wrong
password, the owner saw a session-expired message.

**Why.** The server answered 401, and the app handles every 401 by refreshing its token and
retrying.

**What I did.** A wrong password on that endpoint answers 403, which the app shows as "that
password is wrong".

### The notification question came too early

**What went wrong.** The app asked for permission to send notifications right after
sign-in, before the person had seen anything worth being notified about. An iPhone asks
only once, so a "no" at that moment was final.

**What I did.** The app waits until the account has a device: the first one set up, or a
site joined that has some.

### The iOS build broke when crash reporting was added

**What went wrong.** Adding the crash-reporting package stopped the iPhone build before it
compiled anything.

**Why.** Each Firebase package pins one exact version of the underlying iOS SDK, and two
packages wanted different versions.

**What I did.** Moved the Firebase packages up together as a set, and raised the minimum
versions so a later update can't quietly pick a mismatched pair again.

### Store reviewers have no hardware

**The problem.** Apple and Google review the app without owning a 4Tech device, and an
empty app gets rejected.

**What I did.** Two things. A demo mode inside the app, open from the sign-in screen, that
runs the real screens with sample devices and sends nothing to the server. And simulated
devices on the server: a review account whose devices answer commands like real ones. The
tool that removes a review account refuses to touch an account without simulated devices,
so it can't be pointed at a real customer by mistake.

---

## The IoT server

### Readings overwrote each other

**What went wrong.** When a device sent readings quickly, some updates were lost.

**Why.** The server read a device's stored state, merged the new reading and wrote it back.
Two messages arriving close together both read the old state, and the second write erased
the first.

**What I did.** One atomic "insert or update" statement in the database. I also limited the
"last seen" update to one write per device per 30 seconds, and replaced a row-by-row
time-out check with a single statement.

### The database session committed half-finished work

**What went wrong.** The shared database session committed automatically on the way out of
every request, including requests that had failed partway.

**What I did.** Requests commit explicitly when their work is complete; the shared code
only rolls back and closes. In the same change the database got real migrations in place
of a hand-run SQL file, and indexes on every column used for lookups.

### Background jobs must run exactly once

**The problem.** The API starts jobs when it boots: the MQTT subscriber (with a fixed
client ID), the broker setup check, and the clean-up workers. With several worker
processes, each would start its own copy, and the copies would fight.

**What I did.** The API runs as a single worker, and that decision is written down where
the next person will look for it. On the website, where a scheduled job had to survive
several workers, I used a Postgres advisory lock so that only one of them runs it.

---

## The website

### Orders failed on the real database while every test passed

**What went wrong.** Creating an order failed on Postgres.

**Why.** Order numbers came from a database sequence that was defined in the code but never
attached to the schema, so it was never created. In Postgres a failed statement aborts the
whole transaction, so order creation went down with it. The tests ran on SQLite, which has
no sequences, and missed it.

**What I did.** Attached the sequence to the schema, and made the order-number code handle
both databases.

### Contact messages were being lost, twice over

**What went wrong.** Long messages from the contact form failed with a server error, and
the lead was gone. And with email delivery not configured, the messages that did arrive
were stored where nobody could see them.

**Why.** A "no duplicates" rule on the messages table used an index type with a size limit;
a long message exceeded it. The rule had never caught a duplicate anyway. And there was no
admin screen for messages at all.

**What I did.** Removed the rule, and built a Messages screen in the admin panel with a
badge for unanswered ones. The notification email is now sent after the visitor gets their
answer, so a slow email service can't hold the form up.

### Phone numbers that real people type were rejected

**What went wrong.** "079 331 3220", "+962 79 …" and numbers typed with Arabic-Indic digits
were all refused with a generic error. Numbers with a leading zero produced WhatsApp links
that opened dead chats.

**What I did.** Numbers are normalized before they're checked, and a leading 0 is read as a
Jordanian number when a WhatsApp link is built.

### Confirmation messages gave the wrong time

**What went wrong.** The WhatsApp message confirming a workshop seat gave the start time in
UTC, three hours off for a student in Amman.

**What I did.** Times are given in Amman time. The slim server image had no time zone data,
so I added it as a dependency.

### The public API was showing private meeting links

**What went wrong.** The public workshop list included each online workshop's meeting link,
so anyone could join without registering. It also kept listing workshops that had ended.

**What I did.** Neither any more. A test covers each.

### AI answers arrived all at once

**What went wrong.** The assistant's answers should appear word by word. In production each
answer arrived in one piece at the end.

**Why.** The request passes through two proxies, and a proxy that buffers or compresses a
response holds it until it's complete.

**What I did.** The chat endpoint has its own proxy rule with buffering off, and passes a
header on to the proxy in front so that it doesn't buffer either.

### A price the visitor's browser could fake

**The problem.** The specification said to never trust a price from the browser, and also
to keep the server from ever touching the model file. But then the volume the price is
computed from comes from the browser, and a fake small volume gives a cheap quote.

**What I did.** The server checks that the volume is plausible for the model's dimensions
and rejects absurd values. Suspicious quotes and models with holes are marked for manual
review, and staff check the real file before confirming payment. I documented it as a
known, bounded risk instead of pretending it was solved. It is one of 18 contradictions and
gaps in the specification that I found, decided and wrote down.

### Demo data could reach the live site

**What went wrong.** The script that creates the first admin account also added sample
products, sample workshops and a joke "featured video". Run on the live server, it would
have published them.

**What I did.** Demo data is opt-in, and refused in production. So is the default admin
password.

---

## Deployment and operations

### A deploy could leave the site broken

**What I did.** The deploy script validates the configuration, backs up the database and
runs migrations before it switches anything. Then it tests the site, the admin panel and
the API through the proxy. If any test fails, it rolls back to the previous version by
itself. Backups also run nightly, are verified, and can be copied off the server.

### A stopped backend took the whole site down

**What went wrong.** If the API container was down, the static website and the admin panel
went down with it, although neither needs the API to be served.

**Why.** The proxy looked up the addresses of the services behind it once, at startup.

**What I did.** The proxy resolves them at request time. A stopped backend now costs only
the API.

### Containers flapping "unhealthy" while working fine

**What went wrong.** Health checks reported containers as unhealthy although they were
serving requests.

**Why.** The small Alpine images resolve `localhost` to the IPv6 address, and the services
were listening on IPv4 only.

**What I did.** Health checks use `127.0.0.1`. The same fault appeared in both projects.

### The certificate script failed on its first real run

**What went wrong.** Two faults when issuing the API's Let's Encrypt certificate. The
script restarted nginx from the base configuration alone, which quietly removed every
container's memory limit. Then the certificate tool refused to run.

**Why.** The script writes a temporary self-signed certificate so nginx can start, and the
tool will not adopt a certificate folder it didn't create.

**What I did.** The script names both configuration files, removes the placeholder just
before asking for the real certificate, and no longer forces a new certificate on every
run, which had been using up Let's Encrypt's rate limit for nothing.

### Links lost their port behind the proxy

**What went wrong.** Locally, opening `/en/services` redirected to an address without the
port, which went nowhere.

**Why.** nginx's automatic trailing-slash redirect builds a full address from the internal
host name, which has no public port.

**What I did.** Relative redirects, and sub-pages served directly with no redirect at all.

### Fitting a platform into 2 GB

**The problem.** A database, an MQTT broker, two APIs, a website, an admin panel and
monitoring, all on one server with 2 GB of RAM.

**What I did.** A memory limit for every container, a database tuned for the size, a
website made of static files, an admin panel with no build step in a 5 MB container, swap
as a safety margin, and a written plan for when to move to a bigger server.
