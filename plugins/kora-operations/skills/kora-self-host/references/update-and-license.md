# Updates, Rollback, And License Operations

## Online Updates

```sh
./koractl update latest
./koractl update 0.2.0
```

Online update downloads a release bundle from the configured release base
URL, verifies the checksum, applies bundle-owned files, and restarts Kora.

For offline or controlled Enterprise updates, provide the bundle directly and
pass the checksum file when available:

```sh
./koractl update --bundle ./kora-platform-deploy-0.2.0.tar.gz --checksum ./kora-platform-deploy-0.2.0.tar.gz.sha256
```

The update command backs up `.env`, `license/`, and `state/`, preserves
operator-owned files, replaces bundle-owned files, updates `KORA_IMAGE_TAG`
from the target bundle, then starts Kora. If startup fails, it prints the
backup directory and manual rollback steps.

## Rollback (Manual In V1)

1. restore the previous bundle files from the backup directory
2. restore the previous `.env` if needed
3. set the previous `KORA_IMAGE_TAG`
4. run `./koractl start`

Do not assume automatic database rollback; there is none.

## License Operations

```sh
./koractl license status
./koractl license activate --install-session kora_ins_...
./koractl license install-file --license ./license.json --keys ./license-public-keys.json
```

`license activate` claims an online install session and writes
`license/license.json`, `license/license-public-keys.json`,
`license/license-online-state.json`, and `license/license-deployment-token`.
`license install-file` is the offline path for out-of-band license material.

## Deployment Token Hygiene

`license/license-deployment-token` is a bearer secret used by Platform API
check-ins and by the Platform worker only to inject online account context
into granted extension handlers. Keep it owner-readable only. Do not log it,
commit it, or send it to support.

## Offline Image Handling

Air-gapped image delivery is an Enterprise packaging concern, not a public
installer prompt or an update flag. The customer-facing path is a prepared
offline package with its own instructions.
