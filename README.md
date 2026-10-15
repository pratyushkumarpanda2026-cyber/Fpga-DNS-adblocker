# fpga-dns-adblocker

A DNS ad blocker described as digital hardware in Verilog. No CPU and no software: a streaming circuit reads each DNS query as it passes, decides whether the domain is on a blocklist, and either blocks it or lets it through.

**Live explainer:** https://claude.ai/artifact/W6kpz4z93wr5oawMi7zq36 (step-by-step animation of the circuit)

## How it works

```
DNS query bytes (1 per clock)
        |
  [ dns_parser ]      skips the 12-byte header, turns length-prefixed labels
        |             (0B "doubleclick" 03 "net" 00) back into doubleclick.net
        v
  [ domain_filter ]   FNV-1a hash updated per character, then a 2-bit
        |             Bloom filter lookup in a 64K-bit memory
        v
  blocked = 1  -> reply 0.0.0.0 (ad never loads)
  blocked = 0  -> forward to the real DNS server
```

The hash is computed while the name streams in, so the decision is ready the moment the name ends.

## Files

| File | Purpose |
|---|---|
| `rtl/dns_parser.sv` | FSM that extracts the domain name from a DNS packet |
| `rtl/domain_filter.sv` | Streaming FNV-1a hash and Bloom filter lookup |
| `tb/tb_dns.sv` | Testbench: builds DNS packets byte by byte, teaches the filter one ad domain, then checks it |

## Run the simulation

**Online (no install):** open [EDA Playground](https://edaplayground.com), choose Icarus Verilog 12.0, paste the testbench in the left box and both RTL modules in the right box, then click Run.

**Locally:**
```bash
iverilog -g2012 -o sim rtl/domain_filter.sv rtl/dns_parser.sv tb/tb_dns.sv
vvp sim
gtkwave dump.vcd
```

Expected output:
```
doubleclick.net -> blocked = 1 (expect 1)
google.com      -> blocked = 0 (expect 0)
```

## Limitations

- Only sees plain DNS. Encrypted DNS (DoH/DoT) and ads served from the site's own domain are not affected.
- Bloom filters can give rare false positives, but never false negatives.
- The full name is hashed, so subdomains (such as `ads.doubleclick.net`) need their own entry or suffix hashing.
- The `learn` input fills the filter in simulation. On real hardware the table would be loaded from a file generated on a PC.

## Roadmap

- [x] DNS question parser
- [x] Streaming hash and Bloom filter
- [ ] Ethernet / IPv4 / UDP header parser (port 53 detection)
- [ ] Response generator (fake `0.0.0.0` answer, IP checksum)
- [ ] Suffix matching for subdomains
- [ ] Run on an FPGA board with two Ethernet ports
