---
date: 2023-06-21T11:02:59Z
title: Synchorniser feature with MDCB
tags: ["High Availability", "Synchroniser"]
description: "Synchorniser feature with MDCB"
menu:
  main:
    parent: "Ensure High Availability"
weight: 7
---

## Overview

In order to process API requests successfully the worker Gateways need resources such as API keys, certificates, and OAuth clients.

Tyk Gateway v4.1 introduces an improved synchroniser functionality within Multi Data Centre Bridge (MDCB) v2.0.3. Prior to this release, the API keys, certificates and OAuth clients required by worker Gateways were synchronised from the controller Gateway on-demand. With Gateway v4.1 and MDCB v2.0.3 we introduce proactive synchronisation of these resources to the worker Gateways when they start up.

This change improves resilience in case the MDCB link or controller Gateway is unavailable, because the worker Gateways can continue to operate independently using the resources stored locally. There is also a performance improvement, with the worker Gateways not having to retrieve resources from the controller Gateway when an API is first called.

Changes to keys, certificates and OAuth clients are still synchronised to the worker Gateways from the controller when there are changes and following any failure in the MDCB link.

### How does worker Gateways get resources from MDCB control plane

**Without Synchroniser**

If Synchroniser is disabled, the resources were pulled by the worker Gateways on-demand and not in advance. It means that first it checks if the resource lives in the local redis and if it doesn’t exist then it tries to pull it from the management layer to store it locally.

Every time that a key is updated or removed the management layer emits a signal to all the cluster gateways to update the key accordingly.

Considerations:

- In a situation where MDCB or the management redis is down or having issues then the worker gateway would be affected as well, which is not desired.

[image]

**With Synchroniser**

If Synchroniser is enabled, API keys, certificates and OAuth clients are synchronised and stored in the local Redis server in advance, and since dashboard v4.1.0 a signal is emitted when when one of those resources is created, modified or deleted, it allows the worker DCs to respond accordingly, the transmitted information is: type of resource, action (create, update, delete), if hashed (in the case of keys), and resource Id so the changes are applied.

Considerations: 
- Local Redis sizing: if you have a lot of keys / resources to be synchronised, please review sizing of the local redis
- Data residency: Groups will be ignored when synchronizing- all keys (and oauth clients etc) will be propagated to all Redis instances, this might matter for customers who have a single control plane but multiple clusters of worker Gateways connected. All Redises will get All the keys. This has implications if you have data residency requirements.

[image]

### Configuring the Synchroniser for Tyk Self Managed

Synchroniser feature is disabled by default. To enable it, please configure both the worker Gateways and MDCB control plane accordingly.

**Worker Gateway configuration**

First, configure the worker Gateway to enable synchroniser:

`"slave_options":{ "synchroniser_enabled":true }`

Please see [Gateway configuration options](https://tyk.io/docs/tyk-oss-gateway/configuration/#slave_optionssynchroniser_enabled) for reference

Also, if you are running a cluster of Gateways, you must have a GroupID set in order for the Synchronizer to work properly, otherwise keys will not propagate.

`"slave_options":{ "group_id": "FOOBAR" }`

FOOBAR must be unique per-cluster.

Please see [Gateway configuration options](https://tyk.io/docs/tyk-oss-gateway/configuration/#slave_optionsgroup_id) for reference

**MDCB Control Plane configuration**

Then, configure MDCB Control Plane. The most simple configuration to enable this feature via MDCB config file looks like:

`"sync_worker_config":{ "enabled":true }`

In order to configure how often the worker gateways read the signals from MDCB you can use the configuration option in `key_space_sync_interval` which is the interval (in seconds) that they will take to check if there’re any changes, it defaults to 10 seconds.

All the authentication keys created in the management layer are replicated exactly the same to the worker DCs no matter which authentication method is being used: JWT, custom keys, Auth tokens, open ID, mTLS and so on.

In an MDCB environment, the quotas and rates limits are stored locally per worker cluster, it means that they are not synchronised with others worker clusters or the management layer.

Please see [MDCB configuration options](https://tyk.io/docs/tyk-multi-data-centre/mdcb-configuration-options/#sync_worker_config) for reference

If API keys were used and hash key is disabled, please also set these additional configurations on following components:

- MDCB:

`"sync_worker_config":{ "enabled":true, "hash_keys": false }, "hash_keys": false` 

- Dashboard:

`"hash_keys": false` 

- Management Gateway:

`"slave_options":{ "synchroniser_enabled": true }, "hash_keys": false` 

If certificates were used, please also set these additional configurations:

- MDCB

Set `"security.private_certificate_encoding_secret"` with the certificate encoding secret. This is required because MDCB would decode the certificate first before propagating it to worker gateways. The worker Gateways could encode the certificate with their own secret.

Please see [MDCB configuration options](https://tyk.io/docs/tyk-multi-data-centre/mdcb-configuration-options/#securityprivate_certificate_encoding_secret) for reference

### Configuring the Synchroniser for Tyk Cloud

