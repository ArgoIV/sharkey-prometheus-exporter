# Deployment Guide

## Prerequisites

- A running Sharkey/Misskey instance
- `bash`, `curl`, `jq` on the host
- One of:
  - Prometheus with node_exporter (textfile collector enabled)
  - Grafana Alloy with `prometheus.exporter.unix` textfile support

## Install

```bash
# Clone the repo
git clone https://github.com/SiteRelEnby/sharkey-prometheus-exporter.git
cd sharkey-prometheus-exporter

# Copy exporter to a system location
cp sharkey-exporter.sh /usr/local/bin/
chmod +x /usr/local/bin/sharkey-exporter.sh

# Create output directory
mkdir -p /var/lib/prometheus-textfile
```

## Configure cron

### Basic metrics (no auth)

```bash
# Add to crontab
(crontab -l 2>/dev/null; echo '* * * * * /usr/local/bin/sharkey-exporter.sh --output /var/lib/prometheus-textfile/sharkey.prom') | crontab -
```

### With admin metrics

```bash
# First, see exactly which permissions to enable:
sharkey-exporter.sh --create-token

# Then create the token in your Sharkey web UI (Settings > API > Generate Access Token)
# and save it to a file:
mkdir -p /etc/sharkey-exporter
echo 'YOUR_TOKEN_HERE' > /etc/sharkey-exporter/token
chmod 600 /etc/sharkey-exporter/token

(crontab -l 2>/dev/null; echo '* * * * * /usr/local/bin/sharkey-exporter.sh --token-file /etc/sharkey-exporter/token --output /var/lib/prometheus-textfile/sharkey.prom') | crontab -
```

Note: Sharkey's token creation API is restricted to browser sessions, so token
creation can't be automated from the CLI. The `--create-token` flag prints which
permissions to tick — only 4 read-only scopes are needed.

### Custom instance URL

If the exporter runs on a different host, or Sharkey listens on a non-default port:

```bash
sharkey-exporter.sh --instance https://your.instance.tld --output /var/lib/prometheus-textfile/sharkey.prom
```

## Grafana Alloy setup

Add to your Alloy config:

```alloy
// Sharkey metrics via textfile collector
prometheus.exporter.unix "sharkey_textfile" {
  textfile {
    directory = "/var/lib/prometheus-textfile"
  }
  disable_collectors = ["arp","bcache","bonding","btrfs","conntrack","cpu","cpufreq",
    "diskstats","dmi","edac","entropy","fibrechannel","filefd","filesystem","hwmon",
    "infiniband","ipvs","loadavg","mdadm","meminfo","netclass","netdev","netstat",
    "nfs","nfsd","nvme","os","powersupplyclass","pressure","rapl","schedstat",
    "selinux","sockstat","softnet","stat","tapestats","thermal_zone","time",
    "timex","udp_queues","uname","vmstat","watchdog","xfs","zfs"]
}

prometheus.scrape "sharkey_metrics" {
  targets         = prometheus.exporter.unix.sharkey_textfile.targets
  forward_to      = [prometheus.relabel.sharkey_labels.receiver]
  scrape_interval = "60s"
}

prometheus.relabel "sharkey_labels" {
  forward_to = [prometheus.remote_write.your_endpoint.receiver]

  rule {
    target_label = "job"
    replacement  = "sharkey"
  }

  rule {
    target_label = "instance"
    replacement  = "your.instance.tld"
  }
}
```

## Prometheus node_exporter setup

Enable the textfile collector:

```bash
node_exporter --collector.textfile.directory=/var/lib/prometheus-textfile
```

## Verify

```bash
# Run manually
sharkey-exporter.sh --output /tmp/test.prom
cat /tmp/test.prom

# Check for valid Prometheus format
promtool check metrics < /tmp/test.prom
```
