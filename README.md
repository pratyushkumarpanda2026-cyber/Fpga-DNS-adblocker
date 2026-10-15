# Fpga-DNS-adblocker
A DNS ad blocker described as digital hardware in Verilog. No CPU and no software: a streaming circuit reads each DNS query as it passes, decides whether the domain is on a blocklist, and either blocks it or lets it through.
