---
layout: doc_worker_add
title: Rspamd proxy worker
---

# Rspamd proxy worker

This worker provides different functionality to build multiple layers systems and to handle Milter protocol. Here is a short list of functions provided by proxy worker:

* Forwarding of messages to scanning layer
* Directly interacting with MTA using Milter protocol
* Perform load balancing, retransmitting and health checks for scanning layer
* Add encryption and/or compression to the scan requests
* Mirror some portion of traffic to some test server
* Compare results of mirrored request
* Perform messages scan by own (self-scan mode)

## Milter support

From Rspamd 1.6, rspamd proxy worker supports `milter` protocol which is supported by some of the popular MTA, such as Postfix or Sendmail. The introducing of this feature also finally obsoletes the [Rmilter](https://rspamd.com/rmilter/) project in honor of the new integration method. Milter support is presented in `rspamd_proxy` **only**, however, there are two possibilities to use milter protocol:

* Proxy mode (for large instances) with a dedicated scan layer
* Self-scan mode (for smal instances)

### Self-scan mode

<img class="img-responsive" src="{{ site.baseurl }}/img/rspamd_milter_direct.png">

In this mode, `rspamd_proxy` scans messages itself and talk to MTA directly using Milter protocol. The advantage of this mode is its simplicity. Here is a samle configuration for this mode:

~~~ucl
# local.d/worker-proxy.inc
milter = yes; # Enable milter mode
timeout = 120s; # Needed for Milter usually
upstream "local" {
  default = yes; # Self-scan upstreams are always default
  self_scan = yes; # Enable self-scan
}
~~~

### Proxy mode

<img class="img-responsive" src="{{ site.baseurl }}/img/rspamd_milter_proxy.png">

In this mode, there is a dedicated layer of Rspamd scanners with load-balancing and optional encryption and/or compression. For this setup, the configuration might be different. Here is a small sample of proxy mode with 4 scanners where 2 scanners are more powerful and receive more requests:

~~~ucl
# local.d/worker-proxy.inc
milter = yes; # Enable milter mode
timeout = 120s; # Needed for Milter usually

upstream "scan" {
  default = yes;
  hosts = "round-robin:host1:11333:10,host2:11333:10,host3:11333:5,host4:11333:5";
  key = "..."; # Public key for encryption, generated by rspamadm keypair (optional)
  compression = yes; # Use zstd compression (optional)
}
~~~

## Mirroring

<img class="img-responsive" src="{{ site.baseurl }}/img/rspamd-testing.jpg">

Proxy can be used to test:

* new versions of Rspamd;
* new plugins;
* new rules;
* configuration changes;
* ML models.

In this mode, Rspamd mirrors some portion of traffic to a test cluster. Results of their scans are ignored when returning reply to a client, however, optional compare scripts could be started to evaluate mirror results. Here is a sample configuration of this setup (we are not using milter mode in this sample):

~~~ucl
# local.d/worker-proxy.inc
# Main scan layer
upstream "scan" {
  default = "yes";
  hosts = "round-robin:host1:11333:10,host2:11333:10,host3:11333:5,host4:11333:5";
  key = "..."; # Public key for encryption, generated by rspamadm keypair
  compression = yes; # Use zstd compression
}

mirror "test" {
  hosts = "test:11333";
  probability = 0.1; # Mirror 10% of traffic
  key = "..."; # Public key for encryption, generated by rspamadm keypair
  compression = yes; # Use zstd compression
}
~~~

### Compare scripts

Compare scripts are intended to perform some simple actions with the results returned by a mirror and main cluster machine. They cannot do async request, so all you can do is to write to logs or to files. Here is a simple example of such a script in the configuration:

~~~ucl
# local.d/worker-proxy.inc
  script =<<EOD
return function(results)
  local log = require "rspamd_logger"

  for k,v in pairs(results) do
    if type(v) == 'table' then
      log.infox("%s: %s", k, v['score'])
    else
      log.infox("err: %s: %s", k, v)
    end
  end
end
EOD;
~~~