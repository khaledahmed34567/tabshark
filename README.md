# TabShark

**A browser-native PCAP dissector.** Open a `.pcap` or `.pcapng` capture, filter it, and inspect every protocol layer down to the individual bytes. Everything runs on the page: no backend, no external APIs, no uploads, no AI at runtime.

> Phase 1 of 3: Web application. Phase 2 is the Chrome extension and Phase 3 is the desktop app.

![TabShark](assets/logo.png)

---

## Quick start

TabShark is plain HTML, CSS and JavaScript with no build step and no dependencies.

1. Open `index.html` in any modern browser (Chrome, Edge, Firefox, Safari). Double-clicking the file is enough.
2. Click **Load Sample Capture** to explore immediately, or drag a `.pcap` / `.pcapng` file onto the page.

To serve it locally instead:

```bash
python3 -m http.server 8080
# then visit http://localhost:8080
```

## Features

| Area | What you get |
| --- | --- |
| Capture formats | `.pcap` (microsecond and nanosecond, big and little endian) and `.pcapng` (SHB, IDB, EPB, SPB, per-interface timestamp resolution) |
| Link layers | Ethernet, Raw IP, BSD loopback, Linux cooked capture |
| Protocols | Ethernet II, 802.1Q, ARP, IPv4, IPv6 (with extension headers), TCP, UDP, ICMP, ICMPv6, DNS, HTTP/1.x, TLS handshake detection with SNI |
| TCP detail | Flags (SYN, ACK, FIN, RST, PSH, URG, ECE, CWR, NS) shown bit by bit, raw and relative sequence and acknowledgment numbers, window size, options (MSS, window scale, SACK, timestamps) |
| Analysis | Checksum verification for IPv4, TCP, UDP and ICMP, retransmission detection, reset detection, DNS error codes, malformed-packet detection |
| Workspace | Packet list, protocol tree and hex/ASCII viewer, all kept in sync |
| Filtering | Live display filter, quick-filter chips, and a find box that searches the list and payload text |
| Sample capture | Built in memory on demand: ARP, DNS (success and NXDOMAIN), TCP handshake, HTTP GET and 200 OK, UDP, ICMP echo, TLS with a retransmitted Client Hello, and a TCP reset |

### The three synchronized panes

1. **Packet list.** Virtualized, so large captures scroll smoothly. Columns: No., Time, Source, Destination, Protocol, Length, Info. Rows are tinted by protocol; errors are tinted red.
2. **Protocol tree.** Collapsible, every layer and bit-level flag of the selected packet.
3. **Hex and ASCII.** Offset, hexadecimal bytes and ASCII side by side.

**Cross-highlighting works both ways.** Click any layer or field in the tree and its exact byte range lights up in the hex and ASCII view. Click or hover a byte and the tree jumps to the field that owns it.

## Display filter syntax

The filter is a small Wireshark-style language. It is validated as you type; a red outline and message explain any mistake.

| Example | Meaning |
| --- | --- |
| `tcp`, `udp`, `dns`, `http`, `tls`, `icmp`, `arp`, `ip`, `ipv6` | Packets containing that protocol |
| `error` | Resets, retransmissions, malformed packets, DNS errors, ICMP unreachable |
| `ip.addr == 192.168.1.10` | Source or destination IPv4 address |
| `ip.src == 10.0.0.1`, `ip.dst != 10.0.0.1` | Source or destination only |
| `tcp.port == 80`, `udp.port == 53`, `port == 443` | Source or destination port |
| `tcp.flags.syn && !tcp.flags.ack` | Connection attempts |
| `tcp.flags.rst` | Resets |
| `tcp.analysis.retransmission` | Retransmitted segments |
| `frame.len > 200` | Packets longer than 200 bytes |
| `dns.qry.name contains "example"` | DNS queries by name |
| `http.request.method == "GET"` | HTTP requests by method |
| `tls.handshake.extensions_server_name contains "example"` | TLS by SNI |
| `(tcp \|\| udp) && !dns` | Grouping and negation |
| `example.com` | Any bare word searches source, destination, protocol and info |

Operators: `==`, `!=`, `>`, `<`, `>=`, `<=`, `contains`. Combine with `&&` / `and`, `||` / `or`, `!` / `not`, and parentheses.

Other supported fields: `eth.addr`, `eth.src`, `eth.dst`, `ipv6.addr`, `ip.ttl`, `tcp.srcport`, `tcp.dstport`, `tcp.len`, `tcp.seq`, `tcp.ack`, `tcp.window_size`, `tcp.flags.fin|syn|rst|psh|ack|urg`, `udp.srcport`, `udp.dstport`, `http.host`, `http.response.code`, `frame.number`, `frame.cap_len`, `frame.time_relative`.

## Keyboard shortcuts

| Key | Action |
| --- | --- |
| `/` | Focus the display filter |
| `Up` / `Down`, `PageUp` / `PageDown`, `Home` / `End` | Move through the packet list |
| `Right` / `Left` / `Up` / `Down` in the tree | Expand, collapse and move between fields |
| `Enter` / `Shift+Enter` in Find | Next and previous match |
| `?` | Open the How to Use guide |
| `Esc` | Clear the filter or find box, or close the guide |

## Project structure

```
tabshark/
  index.html        Markup: splash, workspace, onboarding modal
  styles.css        Design tokens, layout, animations
  pcap-parser.js    Binary reader, dissector, flow analysis, sample generator
  app.js            UI: filter engine, virtual list, tree, hex viewer, onboarding
  README.md
  assets/           logo.png (transparent), icon.png (black background), favicon.png
  fonts/            Azer (Regular, Medium, Bold) and Azal Display
```

### How the parser is organized

`pcap-parser.js` exposes one global, `window.TabShark`:

- `readCapture(arrayBuffer)` detects the format from its magic number and returns the packets as `Uint8Array` views over the original buffer, so no bytes are copied.
- `dissect(packet, buildTree)` decodes one packet with `DataView`. A fast summary pass fills the list columns; the full pass builds the protocol tree. Every tree node records the byte range it covers, which is what drives the cross-highlighting.
- `FlowTracker` walks packets in order to compute relative TCP numbers and detect retransmissions.
- `generateSample()` writes a genuine binary capture into an `ArrayBuffer`, with valid IPv4, TCP and UDP checksums.

## Privacy

TabShark makes no network requests. Fonts, icons and scripts are all local files. Capture files are read with the browser's File API and never leave the page.

## Visual identity

| Token | Value |
| --- | --- |
| Background canvas | `#121212` |
| Surface and panels | `#1E1E1E` |
| Elevated and hover | `#262626` |
| Borders | `#2E2E2E` |
| Primary accent (Electric Teal) | `#14B8A6` |
| Secondary accent (Sky Cyan) | `#38BDF8` |
| Text | `#F3F4F6` and `#9CA3AF` |

Protocol colors: TCP `#A78BFA`, UDP `#60A5FA`, HTTP/HTTPS/TLS `#34D399`, DNS `#2DD4BF`, ICMP/ARP `#FBBF24`, errors `#F87171`.

All icons are inline SVG with a 1.5px stroke. The interface contains no emoji.

### Brand assets

- `assets/logo.png` is the wordmark on a transparent background, used in the top bar and splash screen.
- `assets/icon.png` is the wordmark on a black background, used as the favicon source, touch icon and in the guide.
- `fonts/` holds the 29LT Azer family (Regular, Medium, Bold) for interface text and 29LT Azal Display for headings. Confirm that your license for these fonts covers web embedding and redistribution before publishing.

## Limitations

- No TCP stream reassembly: an HTTP message or TLS record split across segments is shown per segment.
- TLS is detected and summarized (versions, SNI, cipher suites, extensions); encrypted payloads are not decrypted.
- HTTP/2, QUIC, DTLS and application protocols beyond those listed appear as plain TCP or UDP with a data preview.
- Very large captures are parsed on the main thread in chunks with a progress bar. A Web Worker version is a candidate for a later phase.

## Roadmap

| Phase | Deliverable | Status |
| --- | --- | --- |
| 1 | Web application | Delivered |
| 2 | Chrome extension (Manifest V3, popup launcher, full-tab workspace) | Awaiting confirmation |
| 3 | Desktop application (native file dialogs, `.pcap` file association) | Awaiting confirmation |

## Author

Built by **Khaled Ahmed**.

LinkedIn: [linkedin.com/in/khaled-ahmed-241002386](https://www.linkedin.com/in/khaled-ahmed-241002386)
